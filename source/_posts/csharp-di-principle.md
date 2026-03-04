---
title: C# 依赖注入（DI）原理
date: 2026-03-04 16:31:00
tags: [C#, 设计模式, 依赖注入, 后端]
categories: 技术笔记
---

## 核心思想

DI 容器本质是一个对象工厂：你告诉它「接口对应哪个实现」，它负责自动创建实例并注入依赖。

## 完整代码

```csharp
DIContainer con = new DIContainer();
con.Register<IUserRepository, UserRepository>(); // 注册：IUserRepository → UserRepository
con.Register<IUserService, UserService>();       // 注册：IUserService → UserService
var service = con.Resolve<IUserService>();       // 解析：自动创建并注入依赖
Console.WriteLine(service.SayHi());             // 输出：hi
```

---

## 第一部分：注册（Register）

```csharp
public void Register<TInterface, TImplementation>()
    where TImplementation : TInterface
{
    _typeMap[typeof(TInterface)] = typeof(TImplementation);
}
```

做了什么：往字典里存一条映射「接口类型 → 实现类型」。

- `where TImplementation : TInterface`：泛型约束，要求实现类必须实现该接口，防止注册不相关的类型。编译期检查，写错直接报错。
- `typeof()`：获取类型的 `Type` 对象，是运行时的类型元数据，用作字典的 key。

---

## 第二部分：解析（Resolve）

```csharp
public T Resolve<T>()
{
    return (T)Resolve(typeof(T));
}
```

为什么需要两个 Resolve：泛型版本是给外部调用的，返回具体类型不用强转。内部递归解析时类型是运行时变量（Type），泛型不接受变量，所以需要非泛型版本。

---

## 第三部分：核心逻辑（非泛型 Resolve）

```csharp
object Resolve(Type type)
{
    if (!_typeMap.ContainsKey(type))
        throw new Exception($"未注册此类型：{type}");

    if (_instanceMap.ContainsKey(type))
        return _instanceMap[type];

    var implType = _typeMap[type];
    var paras = implType.GetConstructors().First().GetParameters()
        .Select(item => Resolve(item.ParameterType))
        .ToArray();
    var instance = Activator.CreateInstance(implType, paras);
    _instanceMap[type] = instance;
    return instance;
}
```

### 逐行解释第③步

- `_typeMap[type]`：从字典查出实现类型。传入 typeof(IUserService)，得到 typeof(UserService)。
- `GetConstructors().First()`：通过反射获取实现类的第一个公共构造函数。
- `.GetParameters()`：获取构造函数的参数列表。
- `.Select(item => Resolve(item.ParameterType))`：对每个参数递归调用 Resolve，这是 DI 自动注入的关键。
- `Activator.CreateInstance(implType, paras)`：通过反射动态创建实例。
- `_instanceMap[type] = instance`：缓存实例，下次直接返回（单例）。

---

## 递归解析过程

```
Resolve(IUserService)
│  查 _typeMap → UserService
│  构造函数需要 IUserRepository
│
├── Resolve(IUserRepository)
│     查 _typeMap → UserRepository
│     构造函数无参数，直接创建
│     return new UserRepository()
│
│  拿到 UserRepository 实例
│  return new UserService(userRepoInstance)
```

---

## 两个字典的作用

| 字典 | Key | Value | 用途 |
|---|---|---|---|
| `_typeMap` | 接口类型 | 实现类型 | 知道用哪个类来实现接口 |
| `_instanceMap` | 接口类型 | 对象实例 | 缓存已创建的实例（单例） |

---

## 接口与实现的分离

```csharp
public class UserService : IUserService
{
    readonly IUserRepository userRepository;
    public UserService(IUserRepository userRepository)
    {
        this.userRepository = userRepository;
    }
    public string SayHi() => userRepository.SayHi();
}
```

可以随时替换实现而不改 UserService 的代码：

```csharp
con.Register<IUserRepository, UserRepository>();     // 正式环境
con.Register<IUserRepository, MockUserRepository>(); // 测试环境
```
