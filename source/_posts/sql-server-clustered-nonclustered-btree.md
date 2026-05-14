---
title: SQL Server 中的聚集索引、非聚集索引和 B+ Tree
date: 2026-05-14 15:20:00
categories:
  - 技术笔记
tags:
  - SQL Server
  - 数据库
  - 索引
  - B+ Tree
---

## 先别急着看索引

刚开始理解索引时，很容易直接跳到“聚集索引、非聚集索引、B+ Tree”这些词。这样学会比较累，因为这些概念看起来像数据库强行发明出来的名词。

我觉得更自然的入口是先问一个问题：

```sql
SELECT *
FROM Users
WHERE Email = 'taki@example.com';
```

数据库到底怎么找到这行数据？

如果没有索引，答案很直接：一页一页扫。

SQL Server 里数据不是按“表格视图”那样存的，而是放在数据页里。一个数据页通常是 8KB。表里的行会被塞进很多页：

```text
Users 表

Page 101
  Id=1, Name=Taki, Email=taki@example.com
  Id=2, Name=Tom,  Email=tom@example.com

Page 102
  Id=3, Name=Jack, Email=jack@example.com
  Id=4, Name=Lucy, Email=lucy@example.com

Page 103
  Id=5, Name=Mike, Email=mike@example.com
```

如果数据库不知道 `taki@example.com` 在哪一页，就只能从头扫到尾。表小的时候没什么感觉，表一大，慢的不是 SQL 语句本身，而是它读了太多不该读的数据页。

索引要解决的就是这个问题：让数据库少读页。

## B+ Tree 是一套分层目录

SQL Server 里常见的行存索引，本质上是一棵 B+ Tree。可以先把它理解成一本书的多级目录。

如果书只有 10 页，目录没意义；如果书有 100 万页，你不可能从第一页翻到最后一页找一个词。目录要分层：

```text
根节点
  A - M  -> 中间节点 1
  N - Z  -> 中间节点 2

中间节点 1
  A - D  -> 叶子页 10
  E - H  -> 叶子页 11
  I - M  -> 叶子页 12

叶子页 11
  Email = eva@example.com  -> 数据位置
  Email = frank@example.com -> 数据位置
  Email = grace@example.com -> 数据位置
```

查一个值时，数据库不是从头扫，而是一路缩小范围：

```text
根节点 -> 中间节点 -> 叶子节点 -> 找到目标
```

B+ Tree 的几个特点刚好适合数据库：

- 树比较矮。哪怕数据很多，通常也只要读几层页面。
- 叶子节点有序。等值查询、范围查询、排序都能受益。
- 每个节点能放很多 key。数据库页是块状读取，B+ Tree 能很好地利用这种 IO 模型。

这里要注意一个细节：数据库里的 B+ Tree 不是“一整个数据库一棵树”。更准确地说：

```text
一个 B+ Tree 类型的索引，大致对应一棵树。
```

如果一张表有 1 个主键索引、2 个普通索引，那这张表就可能有 3 棵 B+ Tree。不同索引按不同列排序，服务不同查询。

## 表本身怎么放：Heap 和聚集索引

在 SQL Server 里，表数据常见有两种组织方式：

```text
Heap
Clustered Index
```

Heap 就是没有聚集索引的表。数据行大致是哪里有空间就放哪里，本身没有按某个业务键组织。

```text
Users Heap

Page 101: Id=3, Id=8
Page 102: Id=1, Id=5
Page 103: Id=2, Id=7
```

这不是说 heap 完全没规则，而是对查询来说，它没有一条“按 Id 排好”的主路径。你按 `Id = 7` 查，如果没有索引，还是得扫。

聚集索引就不一样了。

```sql
CREATE CLUSTERED INDEX CX_Users_Id
ON Users(Id);
```

有了聚集索引后，表数据本身会按这棵 B+ Tree 来组织。这里最关键的一句话是：

```text
SQL Server 聚集索引的叶子层就是数据行本身。
```

可以粗略画成这样：

```text
CX_Users_Id

            Root Page
          [ Id <= 3 | Id > 3 ]
             /             \
            /               \
     Leaf Page 101       Leaf Page 102
     Id=1, Name=...      Id=4, Name=...
     Id=2, Name=...      Id=5, Name=...
     Id=3, Name=...      Id=6, Name=...
```

上层页面负责导航，叶子页面就是完整数据行。查 `Id = 5` 时，数据库沿着 `Id` 这棵树走到叶子层，就拿到了整行数据。

这也是为什么一张表只能有一个聚集索引。因为数据行本身只能按一种主要方式组织。你不能让同一份数据既按 `Id` 排，又按 `Email` 排，还按 `CreatedAt` 排。其他排序需求只能交给非聚集索引。

## 主键不一定等于聚集索引

很多人会把主键和聚集索引混在一起，因为 SQL Server 创建主键时，默认经常会创建聚集索引。

比如：

```sql
CREATE TABLE Users (
    Id BIGINT IDENTITY(1,1) NOT NULL,
    Email NVARCHAR(200) NOT NULL,
    CONSTRAINT PK_Users PRIMARY KEY (Id)
);
```

在 SQL Server 里，这个主键默认通常会变成：

```sql
PRIMARY KEY CLUSTERED (Id)
```

但这不是概念上的必然关系。

主键是约束，回答的是：

```text
哪一列能唯一标识一行？
```

聚集索引是存储组织方式，回答的是：

```text
数据行按哪棵 B+ Tree 组织？
```

也可以显式写成非聚集主键：

```sql
CONSTRAINT PK_Users PRIMARY KEY NONCLUSTERED (Id)
```

只是日常 OLTP 表里，用自增 `Id` 做聚集主键很常见，因为它窄、稳定、递增，插入时对页面比较友好。

## 非聚集索引是一条额外查找路径

假设 `Users` 表已经按 `Id` 做了聚集索引，但经常按邮箱查用户：

```sql
SELECT *
FROM Users
WHERE Email = 'taki@example.com';
```

如果只有 `Id` 聚集索引，这个查询还是不舒服。因为数据按 `Id` 排，不按 `Email` 排。数据库没法直接根据邮箱走到某个位置。

这时可以建非聚集索引：

```sql
CREATE NONCLUSTERED INDEX IX_Users_Email
ON Users(Email);
```

这会额外维护一棵按 `Email` 排序的 B+ Tree：

```text
IX_Users_Email

Root / Intermediate Pages
  按 Email 导航

Leaf Pages
  Email = jack@example.com -> Id = 3
  Email = taki@example.com -> Id = 1
  Email = tom@example.com  -> Id = 2
```

这棵树的叶子层不是完整用户数据行，而是索引键加行定位信息。

如果表有聚集索引，非聚集索引叶子层里通常保存的是聚集索引键。上面的例子里就是 `Id`。

所以查询过程变成：

```text
1. 走 IX_Users_Email，找到 Email = taki@example.com
2. 得到聚集索引键 Id = 1
3. 再走 CX_Users_Id，找到完整 Users 行
```

第 3 步通常叫 `Key Lookup`，也就是常说的回表。

如果表是 heap，没有聚集索引，非聚集索引叶子层里保存的就不是聚集键，而是 RID，可以理解成：

```text
FileId + PageId + SlotId
```

也就是“第几个文件、第几页、第几个槽位”。数据库拿这个位置再回到 heap 里取行。

## 回表为什么有时很贵

回表不是一定坏。查一两行时，先走非聚集索引再回表通常很快。

麻烦出现在返回行数很多的时候。

比如：

```sql
SELECT *
FROM Orders
WHERE Status = 'Paid';
```

如果 `Status = 'Paid'` 命中了 80% 的订单，那么数据库即使用了 `IX_Orders_Status`，也要对大量行做 Key Lookup。结果可能是：

```text
先读一遍 Status 索引
再一行一行回聚集索引取完整数据
```

这种随机访问很多时，可能还不如直接扫聚集索引。

所以索引不是“建了就一定用”。优化器会估算成本：如果走索引再回表比扫描还贵，它就可能选择扫描。

这也是为什么低选择性字段不一定适合单独建索引。像 `Gender`、`IsDeleted`、`Status` 这种列，要看数据分布和查询条件，不是看到 `WHERE` 里出现就建。

## 覆盖索引：让查询不用回表

如果查询需要的列都在非聚集索引里，就不用回表。

比如：

```sql
SELECT Id, Email
FROM Users
WHERE Email = 'taki@example.com';
```

`IX_Users_Email` 里有 `Email`，叶子层也有聚集键 `Id`，这个查询可以只读非聚集索引。

如果还要返回 `Name`：

```sql
SELECT Id, Email, Name
FROM Users
WHERE Email = 'taki@example.com';
```

但 `Name` 不在索引里，就要回表。可以用 `INCLUDE` 把它放进叶子层：

```sql
CREATE NONCLUSTERED INDEX IX_Users_Email
ON Users(Email)
INCLUDE (Name);
```

这时可以理解成：

```text
IX_Users_Email 叶子层
  Email = taki@example.com -> Id = 1, Name = Taki
```

`Name` 不参与索引排序，只是跟在叶子层，方便查询直接拿到结果。这就是覆盖索引的基本思路。

覆盖索引很实用，但也不能滥用。`INCLUDE` 列越多，索引越大，写入维护成本也越高。读快了，写会变重。

## B+ Tree 对范围查询也很友好

B+ Tree 不只适合等值查询，也适合范围查询。

比如订单表：

```sql
CREATE INDEX IX_Orders_CreatedAt
ON Orders(CreatedAt);
```

查询：

```sql
SELECT *
FROM Orders
WHERE CreatedAt >= '2026-05-01'
  AND CreatedAt < '2026-06-01';
```

数据库可以先沿着 B+ Tree 找到 `2026-05-01` 附近的叶子页，然后顺着叶子页往后读，直到超过 `2026-06-01`。

```text
2026-04-29 -> 2026-04-30 -> 2026-05-01 -> 2026-05-02 -> ... -> 2026-06-01
                              ↑ 从这里开始读
```

这比扫描全表自然要舒服很多。

排序也是类似道理。如果查询的 `ORDER BY` 正好和索引顺序匹配，数据库可能不需要额外排序。

```sql
CREATE INDEX IX_Orders_UserId_CreatedAt
ON Orders(UserId, CreatedAt DESC);
```

这个索引适合：

```sql
SELECT TOP 20 *
FROM Orders
WHERE UserId = 1001
ORDER BY CreatedAt DESC;
```

因为它先按 `UserId` 分组，再在同一个用户下面按 `CreatedAt DESC` 排好。数据库可以直接拿前 20 条，而不是把这个用户所有订单找出来再排序。

## 聚集键选错会影响所有非聚集索引

SQL Server 里有一个很容易忽略的点：如果表有聚集索引，非聚集索引叶子层会带着聚集键。

这意味着聚集键会出现在很多非聚集索引里。

如果聚集键很宽，比如拿一个很长的字符串做聚集索引，那么每个非聚集索引都会变大。索引页能放下的记录变少，树可能更高，缓存效率也变差。

所以聚集键一般推荐：

- 窄
- 稳定
- 唯一或接近唯一
- 尽量递增

自增整数、`BIGINT IDENTITY` 这类字段经常被用作聚集键，就是因为它们符合这些特征。

随机 GUID 做聚集键就要小心。它不是不能用，而是插入位置太随机，容易造成页分裂和碎片。SQL Server 里如果必须用 GUID，可以考虑 `NEWSEQUENTIALID()` 或者把 GUID 主键设为非聚集，再单独选择合适的聚集键。

## 落到执行计划里看

理解这些概念之后，再看执行计划会容易很多。

看到 `Index Seek`，通常说明优化器能沿着某棵 B+ Tree 缩小范围。它不一定只读一行，但至少不是从头把整张表扫过去。

看到 `Index Scan` 或 `Clustered Index Scan`，说明数据库在按页扫描一段甚至整棵索引。有些 scan 是正常的，比如小表、报表、大范围查询；但如果一个本来应该很精准的查询走了 scan，就要回头看索引列顺序、统计信息和查询条件。

看到 `Key Lookup`，说明非聚集索引先找到了行定位信息，但查询还需要回到聚集索引里取其他列。少量 lookup 没问题，大量 lookup 往往就是慢查询的来源之一。这时可以考虑收窄返回列，或者用 `INCLUDE` 做覆盖索引。

所以索引真正改变的不是 SQL 表面写法，而是数据库为了执行这句 SQL 要读多少页、按什么顺序读、要不要来回跳着读。聚集索引、非聚集索引和 B+ Tree 最后都会落到这个问题上。
