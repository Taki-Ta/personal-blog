---
title: Nginx 负载均衡中的加权轮询算法
date: 2026-04-14 11:05:46
categories:
  - 技术笔记
tags:
  - Nginx
  - 负载均衡
  - Rust
  - 算法
---

## 从普通轮询开始

负载均衡最容易想到的办法是轮询。

```text
A -> B -> C -> A -> B -> C -> ...
```

这个算法很干净：不用关心后端是谁，也不用关心上一轮请求花了多久，只要按顺序往后选就行。如果几台后端机器配置差不多，请求耗时也差不多，它通常够用。

但真实情况很少这么整齐。比如 A 是一台低配机器，B 是一台高配机器，继续让它们一人一半流量，就有点浪费 B，也容易让 A 先扛不住。

这就是权重存在的原因。

```nginx
upstream backend {
    server 127.0.0.1:3001 weight=1;
    server 127.0.0.1:3002 weight=2;
}
```

这里的 `weight=2` 可以先粗略理解成：B 的承载能力按 A 的两倍来算。它不是每一秒都精确保证 1:2，也不是每三个请求一定固定分成 1 个给 A、2 个给 B。它更像一个长期比例，流量足够多的时候，整体分布应该靠近这个比例。

我觉得理解权重时最容易走偏的一点是：它不是实时负载指标。权重只是在告诉调度器“这台机器理论上更能扛”。至于它此刻是不是正在卡 GC、是不是连接数已经很多，那是健康检查、最少连接、响应时间调度要处理的问题。

## 经典加权轮询

经典加权轮询的思路比较机械，但很好理解。

它不是把节点展开成：

```text
A, B, B
```

然后普通轮询。这样当然也能跑，但权重一大，列表会膨胀，而且节点变更时不太优雅。

更常见的写法是维护几个状态：

```text
current_index   当前扫描到的节点下标
current_weight  当前权重阈值
max_weight      所有节点中的最大权重
gcd_weight      所有节点权重的最大公约数
```

假设权重是 `[1, 2]`，最大权重是 `2`，最大公约数是 `1`。算法会先找权重大于等于 `2` 的节点，所以 B 先被选中。下一轮阈值降到 `1`，A 和 B 都有机会被选中。长期跑下来，B 的命中次数会接近 A 的两倍。

这种算法的好处是状态少，实现也直接。下面是一个只保留核心逻辑的 Rust 示例：

```rust
#[derive(Debug)]
struct Peer {
    name: &'static str,
    weight: u32,
    available: bool,
}

#[derive(Debug)]
struct ClassicWeightedRoundRobin {
    current_index: usize,
    current_weight: u32,
    max_weight: u32,
    gcd_weight: u32,
}

impl ClassicWeightedRoundRobin {
    fn new(peers: &[Peer]) -> Self {
        let max_weight = peers.iter().map(|peer| peer.weight).max().unwrap_or(0);
        let gcd_weight = peers
            .iter()
            .map(|peer| peer.weight)
            .reduce(gcd)
            .unwrap_or(0);

        Self {
            current_index: peers.len().saturating_sub(1),
            current_weight: 0,
            max_weight,
            gcd_weight,
        }
    }

    fn select<'a>(&mut self, peers: &'a [Peer]) -> Option<&'a Peer> {
        if peers.is_empty() || self.max_weight == 0 {
            return None;
        }

        loop {
            self.current_index = (self.current_index + 1) % peers.len();

            if self.current_index == 0 {
                self.current_weight = self.current_weight.saturating_sub(self.gcd_weight);

                if self.current_weight == 0 {
                    self.current_weight = self.max_weight;
                }
            }

            let peer = &peers[self.current_index];
            if peer.available && peer.weight >= self.current_weight {
                return Some(peer);
            }

            if !peers.iter().any(|peer| peer.available) {
                return None;
            }
        }
    }
}

fn gcd(a: u32, b: u32) -> u32 {
    if b == 0 {
        a
    } else {
        gcd(b, a % b)
    }
}
```

这段代码里有一个细节：只有 `peer.available` 为真时才返回节点。权重再高，也不应该让一个不可用节点参与调度。权重是容量表达，不是健康检查。

经典加权轮询的问题也在这里：它能保证长期比例，但不一定保证短时间内的请求足够平滑。权重差距越大，这个问题越明显。比如 `{5, 1, 1}`，最终比例是对的，但局部序列可能不太好看，某个高权重节点可能短时间内连续接到多次请求。

## 平滑加权轮询

Nginx 后来用的是 smooth weighted round-robin。这个算法名字里的 “smooth” 很关键，它不是为了改变比例，而是为了让请求分布更平滑。

来看 `{A: 5, B: 1, C: 1}`。最终比例应该是 5:1:1。一个不够平滑的序列可能会让 A 连续出现很多次，而平滑加权轮询会尽量给出类似这样的结果：

```text
A, A, B, A, C, A, A
```

A 还是出现 5 次，B 和 C 还是各出现 1 次，但它们被插进了 A 的请求之间。对于后端来说，这比短时间内被连续打很多下要舒服一些。

平滑加权轮询有三个状态比较重要：

```text
weight             配置里的固定权重
effective_weight   当前有效权重
current_weight     每轮调度时变化的动态权重
```

每次选择节点时，大致做三件事：

1. 每个可用节点的 `current_weight` 加上自己的 `effective_weight`。
2. 选出 `current_weight` 最大的节点。
3. 被选中的节点把 `current_weight` 减去总权重。

写成 Rust 示例大概是这样：

```rust
#[derive(Debug)]
struct SmoothPeer {
    name: &'static str,
    weight: i32,
    effective_weight: i32,
    current_weight: i32,
    available: bool,
}

fn select_smooth(peers: &mut [SmoothPeer]) -> Option<&SmoothPeer> {
    let mut total = 0;
    let mut best_index = None;
    let mut best_weight = i32::MIN;

    for (index, peer) in peers.iter_mut().enumerate() {
        if !peer.available {
            continue;
        }

        peer.current_weight += peer.effective_weight;
        total += peer.effective_weight;

        if peer.current_weight > best_weight {
            best_weight = peer.current_weight;
            best_index = Some(index);
        }

        if peer.effective_weight < peer.weight {
            peer.effective_weight += 1;
        }
    }

    let index = best_index?;
    peers[index].current_weight -= total;
    Some(&peers[index])
}
```

`effective_weight` 这个字段很有意思。它让权重不再只是静态配置。节点失败时，可以临时降低它的有效权重；后面节点恢复稳定，再慢慢把有效权重加回配置权重。这样做比“刚恢复就立刻吃满流量”更稳。

当然，平滑加权轮询也不是白来的。它每次选择都要更新多个节点的动态状态。如果是在并发代理里实现，就要认真处理状态同步。用 `Mutex<Vec<PeerState>>` 会比较直观；如果一上来就追求无锁，代码很容易变得难验证。

## 其他策略放在什么位置

加权轮询适合解决“机器能力不同”的问题，但它不适合解决所有负载问题。

`least_conn` 更适合长连接或者请求耗时差距很大的场景。比如一个请求可能持续 10 秒，另一个请求 50 毫秒就结束，只按请求次数轮询就不太准了。这个时候看当前连接数会更合理。

`ip_hash` 解决的是会话粘性。它让同一个客户端尽量落到同一台后端，适合一些没有完全拆掉本地 session 的老系统。代价是节点变化时映射会变，流量也可能因为客户端分布不均而倾斜。

`hash` 或 `consistent hash` 更像缓存场景里的工具。按 URI、用户 ID 或其他 key 做路由，可以让同一个 key 尽量落到同一台机器。一致性 hash 的价值在于节点增删时，不至于让大部分 key 都重新映射。

`random` 看起来很随意，但在节点很多的时候反而有用。尤其是 “power of two choices” 这种做法：随机挑两个节点，再从里面选负载更低的那个，简单但效果通常不错。

所以我会把这些策略粗略分成几类：

| 想解决的问题           | 更接近的策略                |
| ---------------------- | --------------------------- |
| 后端能力不同           | weighted round-robin        |
| 请求耗时差异大         | least_conn                  |
| 需要会话粘性           | ip_hash                     |
| 缓存命中和 key 稳定性  | hash / consistent hash      |
| 节点多，想降低调度成本 | random / random two choices |

如果只是学习负载均衡，我觉得顺序可以是：先写普通轮询，再写经典加权轮询，然后写平滑加权轮询。写完这三个，再看 `least_conn` 和 hash 类策略，会顺很多。

## 几个容易忽略的点

`weight = 0` 最好不要含糊。要么配置校验时直接拒绝，要么明确表示这个节点不参与调度。Nginx 里有 `down` 可以标记节点不可用，自己实现时也应该有清晰语义。

失败重试也要小心。代理发现某个 upstream 失败后，可以换下一个节点试，但不是所有请求都适合重试。像支付、下单、扣库存这类非幂等请求，如果后端其实已经处理成功，只是响应没回来，代理层再重试一次就可能制造重复操作。

还有一个测试问题：只测“高权重节点收到的请求更多”是不够的。更好的测试是跑完整周期，比如权重 1:2 就跑 300 次，看结果是否接近 100:200；权重 5:1:1 就观察前 7 次的序列，确认平滑加权轮询没有把请求集中到一个节点上。

## 小结

加权轮询真正要表达的是“容量比例”，不是实时负载。经典加权轮询简单，适合先把比例跑通；平滑加权轮询更接近 Nginx 的做法，它让比例不变，但把短时间内的请求分布摊得更均匀。

## 参考资料

- [NGINX Admin Guide: HTTP Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- [Nginx `ngx_http_upstream_module` 官方文档](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
- [Nginx load balancing 文档](https://nginx.org/en/docs/http/load_balancing.html)
- [Nginx 开发邮件列表：smooth weighted round-robin balancing](https://mailman.nginx.org/pipermail/nginx-devel/2012-June/002300.html)
- [Nginx upstream round robin 源码](https://github.com/nginx/nginx/blob/master/src/http/ngx_http_upstream_round_robin.c)
