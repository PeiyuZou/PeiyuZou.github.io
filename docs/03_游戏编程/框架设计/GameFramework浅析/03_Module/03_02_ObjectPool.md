## 整体架构

![对象池管理器结构](object_pool_01.png)
/// caption
对象池管理器结构
///

ObjectPoolManager 是对象池的管理器，其中管理着多个 `TypeNamePair` 和 `ObjectPoolBase` 的映射；

单个 `ObjectPoolBase` 是一个池的抽象，`IObjectPool` 提供对象池的约束，GF 默认实现了一个 `ObjectPool` 池

`ObjectBase` 是池中单个元素的抽象，供上层派生

## ObjectPoolManager

`ObjectPoolManager` 核心的数据结构是一个`TypeNamePair` 和 `ObjectPoolBase` 映射的字典，所以它的职责其实很简单，就是维护所有的池子，我们来看看 `IObjectPoolMangaer` 接口设计，它有很多接口声明，大部分都是一些多态的变体，核心能做的就如下几件事（伪代码）：

``` cs title="IObjectPoolMangaer"
public interface IObjectPoolMangaer
{
    int Count { get; } // 获取对象池数量
    bool HasObjectPool() // 检查是否存在对象池
    IObjectPool<T> GetObjectPool() // 获取对象池（IObjectPool）
    ObjectPoolBase GetObjectPool() // 获取对象池（ObjectPoolBase）
    IObjectPool<T> CreateSingleSpawnObjectPool() // 创建允许单次获取的对象池（IObjectPool）
    ObjectPoolBase CreateSingleSpawnObjectPool() // 创建允许单次获取的对象池（ObjectPoolBase）
    IObjectPool<T> CreateMultiSpawnObjectPool() // 创建允许多次获取的对象池（IObjectPool）
    ObjectPoolBase CreateMultiSpawnObjectPool() // 创建允许多次获取的对象池（ObjectPoolBase）
    DestroyObjectPool() // 销毁对象池
    Release() // 释放全部对象池中的可释放对象
    ReleaseAllUnused() // 释放全部对象池中的所有未使用对象
}
```

所以 `IObjectPoolManager` 的设计其实很简单，就是约束管理维护各种池子。

## IObjectPool 和 ObjectPoolBase

在上面 `ObjectPoolManager` 里有一个很有“工程味”的设计，不管是 Get 还是 Create 一个池子，作者都提供了 `IObjectPool<T>` 和 `ObjectPoolBase` 两种返回。为什么要这样设计？这个问题等效于另一个问题：**为什么要设计“接口 + 抽象类”双层，而不是只用抽象类？**

确实，如果硬要回答能不能只用抽象类，当然是可以的，将接口声明的方法搬到抽象类中作为抽象方法一样地可以约束子类的行为。但 GF 这样设计，本质是为了三点：

- 接口用于“对外抽象能力”，它约束了子类至少要具备哪些能力，模块之间只依赖 `IObjectPool`
- 抽象类用于默认实现模板，它提供一些内置的通用逻辑
- 允许轻量级的实现，抽象类中可能有些成员是没那么必要的，某些情况下是冗余的

举两个例子来说明上述观点：

- UGF 的 `ObjectPoolComponent` 中的 `m_Pools` 类型是 `Dictionary<Type, IObjectPool>`，这将接口化编程体现的很好。第一，它只关心 Pool 有没有 `Acquire()` 和 `Release()`；第二，它减轻了对底层的依赖
- 假如要实现一个极简对象池，它不需要统计、自动回收、复杂策略，那么继承抽象类就被迫带一堆没用的逻辑

这里引出“接口 + 抽象类”的经典分工设计：

- Interface：能力定义层，只关心“能干啥”
- Abstract Class：默认实现层，提供通用成员和部分实现，帮你“少写代码”
- 具体实现类：策略层，决定运作细节

---

总的来说，接口 `IObjectPool` 的设计更加轻量和通用，它声明了一个对象池基本的成员

``` cs title="IObjectPool"
public interface IObjectPool<T> where T : ObjectBase
{
    string Name { get; } // 对象池名称
    string FullName { get; } // 对象池完整名称
    Type ObjectType { get; } // 对象池对象类型
    int Count { get; } // 对象池中对象的数量
    int CanReleaseCount { get; } // 对象池中能被释放的对象的数量
    bool AllowMultiSpawn { get; } // 是否允许对象被多次获取
    bool AutoReleaseInterval { get; set; } // 获取或设置对象池自动释放可释放对象的间隔秒数
    int Capacity { get; set; } // 获取或设置对象池的容量
    int ExpireTime { get; set; } // 获取或设置对象池对象过期秒数
    int Priority { get; set; } // 获取或设置对象池的优先级
    void Register(T obj, bool spawned); // 创建对象
    bool CanSpawn(); // 检查对象
    T Spawn(); // 获取对象
    void Unspawn(T obj); // 回收对象
    void SetLocked(T obj, bool locked); // 设置对象是否被加锁
    void SetPriority(T obj, int priority); // 设置对象的优先级
    bool ReleaseObject(T obj); // 释放对象
    void Release(); // 释放对象池中的可释放对象
    void ReleaseAllUnused(); // 释放对象池中的所有未使用对象
}
```

抽象类 `ObjectPoolBase` 则更体现 GF 对于对象池的要求和设计，它提供了一些对派生类的约束，包含了一点点默认实现：

``` cs title="ObjectPoolBase"
public abstract class ObjectPoolBase
{
    public abstract string Name { get; } // 对象池名称
    public abstract string FullName { get; } // 对象池完整名称
    public abstract Type ObjectType { get; } // 对象池对象类型
    public abstract int Count { get; } // 对象池中对象的数量
    public abstract int CanReleaseCount { get; } // 对象池中能被释放的对象的数量
    public abstract bool AllowMultiSpawn { get; } // 是否允许对象被多次获取
    public abstract bool AutoReleaseInterval { get; set; } // 获取或设置对象池自动释放可释放对象的间隔秒数
    public abstract int Capacity { get; set; } // 获取或设置对象池的容量
    public abstract int ExpireTime { get; set; } // 获取或设置对象池对象过期秒数
    public abstract int Priority { get; set; } // 获取或设置对象池的优先级
    public abstract void Release(); // 释放对象池中的可释放对象
    public abstract void ReleaseAllUnused(); // 释放对象池中的所有未使用对象
    public abstract ObjectInfo[] GetAllObjectInfos(); // 获取所有对象信息
    internal abstract void Update(float elapseSeconds, float realElapseSeconds);
    internal abstract void Shutdown();
}
```

相比 `IObjectPool`，`ObjectPoolBase` 只多声明了三个成员函数，`GetAllObjectInfos` 用于调试和查看对象信息，而 `Update` 和 `Shutdown` 则是生命周期函数，池子的生命周期将会被 Manager 控制

## ObjectPool

`ObjectPool` 是 GF 提供的一个内部实现，我们看五个函数的实现即可窥见大概的逻辑：

- Register(T obj, bool spawned)
- Spawn()
- Unspawn()
- GetAllObjectInfos()
- Update(float elapseSeconds, float realElapseSeconds)
- Shutdown()

前三者是从接口的层面看 GF 怎么实现一个对象池的核心（注册、出池和入池）；而后三者是抽象类层面的实现，看它们可以弄清楚 GF 怎么维护和管理一个池子的生命周期

### 1. Register

``` cs title="注册实现"
/// <summary>
/// 创建对象。
/// </summary>
/// <param name="obj">对象。</param>
/// <param name="spawned">对象是否已被获取。</param>
public void Register(T obj, bool spawned)
{
    if (obj == null)
    {
        throw new GameFrameworkException("Object is invalid.");
    }

    Object<T> internalObject = Object<T>.Create(obj, spawned);
    m_Objects.Add(obj.Name, internalObject);
    m_ObjectMap.Add(obj.Target, internalObject);

    if (Count > m_Capacity)
    {
        Release();
    }
}
```

从注册的逻辑链路中可以发现一共涉及了三种“对象”：`Object<T>`, `ObjectBase`, `ObjectBase.Target`，在一切开始之前，需要先讲清楚这三种对象的设计。

- **`Object<T>`** 是内部用于池对象管理的“壳”，泛型约束了池对象必须从 `ObjectBase` 派生，相当于是对 `ObjectBase` 的 Wrapper，它的职责是：
    - 记录状态（是否在使用中）
    - 记录引用信息（池归属、获取时间等）
    - 控制生命周期（Spawn / Unspawn）
    - 避免业务层直接接触池管理细节
- **`ObjectBase`** 是池元素抽象层，虽然对业务层提供了派生，但它并不是真正的业务对象，而是一个“生命周期协议层”，它规定了池对象必须具备以下行为：
    - public abstract object Target { get; } // 持有真正的业务对象
    - protected internal abstract void OnSpawn(); // 出池时的行为
    - protected internal abstract void OnUnspawn(); // 入池时的行为
    - protected internal abstract void Release(bool isShutdown); // 释放时的行为
- **`ObjectBase.Target`** 是真正的业务对象，这样设计是为了脱离继承链路，让业务对象不用继承 ObjectBase

!!! note "为什么要设计 `ObjectBase.Target` ，而不是直接让业务对象继承 `ObjectBase` ？"
    如果所有业务对象都必须继承自 `ObjectBase`，会有三个问题：

    - 侵入性极强
    - 无法池化第三方类（Unity自带组件/C#的内置容器等）
    - 破坏业务层设计（你可能已经有继承体系）

    这里其实是使用了适配器模式，将任意对象适配成池系统可以管理的对象，将池系统与业务彻底解耦。

用一句话来概括：

> `Object<T>` 管“用没用”，`ObjectBase` 管“怎么用”，`Target` 是“真正被用的东西”

这样的设计当然在性能方面来看是有一点点问题，毕竟多一层包装和一层间接访问，但是换来的是极强的扩展性、复用性以及极低的侵入性，正是这种通用框架所需要的。

---

回到注册的实现，池内部使用了两个核心数据结构：

- **`m_Objects`** 是一对多字典，它存储池元素名字和 Wrapper 的映射关系，允许外部通过名称控制池内元素分组，在其他模块中我们可以看到这样的使用（比如 Resource 和 UI ），但通常的应用场景，一般使用空字符名称，即全部的池元素存同一个 Range 中，对外的抽象概念都是统一的
- **`m_ObjectMap`** 则存储了真实业务对象到 Wrapper 的映射关系，外部仅需关心并可以传入 `ObjectBase` 进行某些操作，内部通过 `ObjectBase.Target` 取得 Wrapper 来完成剩余的逻辑即可

调用完 `Register` 就建立了以三层“对象”为核心的单个池，接下来也就可以投入使用了

### 2. Spawn

``` cs title="出池实现"
/// <summary>
/// 获取对象。
/// </summary>
/// <param name="name">对象名称。</param>
/// <returns>要获取的对象。</returns>
public T Spawn(string name)
{
    if (name == null)
    {
        throw new GameFrameworkException("Name is invalid.");
    }

    GameFrameworkLinkedListRange<Object<T>> objectRange = default(GameFrameworkLinkedListRange<Object<T>>);
    if (m_Objects.TryGetValue(name, out objectRange))
    {
        foreach (Object<T> internalObject in objectRange)
        {
            if (m_AllowMultiSpawn || !internalObject.IsInUse)
            {
                return internalObject.Spawn();
            }
        }
    }

    return null;
}
```

出池逻辑很简单，通过名字获取 Range 段之后再拿到 Wrapper，通过 Wrapper 将 `ObjectBase` 出池，出池时会调用 `ObjectBase.OnSpawn()`

### 3. Unspawn

``` cs title="入池逻辑"
/// <summary>
/// 回收对象。
/// </summary>
/// <param name="target">要回收的对象。</param>
public void Unspawn(object target)
{
    if (target == null)
    {
        throw new GameFrameworkException("Target is invalid.");
    }

    Object<T> internalObject = GetObject(target);
    if (internalObject != null)
    {
        internalObject.Unspawn();
        if (Count > m_Capacity && internalObject.SpawnCount <= 0)
        {
            Release();
        }
    }
    else
    {
        throw new GameFrameworkException(Utility.Text.Format("Can not find target in object pool '{0}', target type is '{1}', target value is '{2}'.", new TypeNamePair(typeof(T), Name), target.GetType().FullName, target));
    }
}
```

入池是通过映射关系拿到 Wrapper，通过 Wrapper 将 `ObjectBase` 入池，入池调用 `ObjectBase.OnUnspawn()`

### 4. GetAllObjectInfos

``` cs title="获取所有对象信息"
/// <summary>
/// 获取所有对象信息。
/// </summary>
/// <returns>所有对象信息。</returns>
public override ObjectInfo[] GetAllObjectInfos()
{
    List<ObjectInfo> results = new List<ObjectInfo>();
    foreach (KeyValuePair<string, GameFrameworkLinkedListRange<Object<T>>> objectRanges in m_Objects)
    {
        foreach (Object<T> internalObject in objectRanges.Value)
        {
            results.Add(new ObjectInfo(internalObject.Name, internalObject.Locked, internalObject.CustomCanReleaseFlag, internalObject.Priority, internalObject.LastUseTime, internalObject.SpawnCount));
        }
    }

    return results.ToArray();
}
```

为每个 Wrapper 构建一个信息对象，返回一个信息对象的数组，这对调试有较大帮助

### 5. Update

``` cs title="轮询逻辑"
internal override void Update(float elapseSeconds, float realElapseSeconds)
{
    m_AutoReleaseTime += realElapseSeconds;
    if (m_AutoReleaseTime < m_AutoReleaseInterval)
    {
        return;
    }

    Release();
}
```

一般的池其实没有设计轮询，所以该方法也不在接口中声明而是在抽象类中定义。这里轮询是为了实现自动释放，这是 GF 提供的一种特性。

### 6. Shutdown

``` cs title="终止逻辑"
internal override void Shutdown()
{
    foreach (KeyValuePair<object, Object<T>> objectInMap in m_ObjectMap)
    {
        objectInMap.Value.Release(true);
        ReferencePool.Release(objectInMap.Value);
    }

    m_Objects.Clear();
    m_ObjectMap.Clear();
    m_CachedCanReleaseObjects.Clear();
    m_CachedToReleaseObjects.Clear();
}
```

这里完成一些对池的成员的释放。

## 总结

GameFramework 对象池的设计及其考究。它通过“接口 + 抽象类”的设计向上层提供接口化，向底层约束内部实现；在池内部设计了三种“对象”，将业务对象和池对象解耦，保证了池对象的生命周期管理，又减少了对业务代码的入侵。它足够通用和灵活，能满足绝大多数使用场景。