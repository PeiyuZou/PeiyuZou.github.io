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

这里有一个很有“工程味”的设计，不管是 Get 还是 Create 一个池子，作者都提供了 `IObjectPool<T>` 和 `ObjectPoolBase` 两种返回。为什么要这样设计？这个问题等效于另一个问题：**为什么要设计“接口 + 抽象类”双层，而不是只用抽象类？**

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

TODO