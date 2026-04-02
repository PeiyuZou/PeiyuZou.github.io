## 结构

![引用池的结构](reference_pool_01.png)
/// caption
引用池的结构
///

引用池结构比较简单，它为每一个类型提供一个单独的池子，每个池子内部通过一个队列来做缓存。

## 接口设计

单个引用池对象是一个 `IReference`，它的设计及其简单，只包含一个清理函数，在回池时调用，派生类自己写逻辑保证回池时状态被重置

``` cs
namespace GameFramework
{
    /// <summary>
    /// 引用接口。
    /// </summary>
    public interface IReference
    {
        /// <summary>
        /// 清理引用。
        /// </summary>
        void Clear();
    }
}
```

## 时序

``` cs title="获得流程"
ClientCode
↓
ReferencePool.Acquire<T>()
↓
var collection = ReferencePool.GetReferenceCollection(typeof(T))
↓
collection.Acquire<T>() // m_UsingReferenceCount++, m_AcquireReferenceCount++
↓
if (Queue has references)
{
    Dequeue from m_References
    Return existing reference
}
else // Queue is empty
{
    Create new T() // m_AddReferenceCount++
    Return new reference
}
↓
return reference to ClientCode
```

``` cs title="释放流程"
ClientCode
↓
ReferencePool.Release(reference)
↓
var collection = ReferencePool.GetReferenceCollection(reference.GetType())
↓
collection.Release(reference)
↓
reference.Clear() // override by client code
↓
if (strict check enabled)
{
    if (Queue.Contains(reference)) throw Exception
}
↓
Enqueue to Queue // m_ReleaseReferenceCount++, m_UsingReferenceCount--
```

## 类型检查

``` cs
private static void InternalCheckReferenceType(Type referenceType)
{
    if (!m_EnableStrictCheck)
    {
        return;
    }

    if (referenceType == null)
    {
        throw new GameFrameworkException("Reference type is invalid.");
    }

    if (!referenceType.IsClass || referenceType.IsAbstract)
    {
        throw new GameFrameworkException("Reference type is not a non-abstract class type.");
    }

    if (!typeof(IReference).IsAssignableFrom(referenceType))
    {
        throw new GameFrameworkException(Utility.Text.Format("Reference type '{0}' is invalid.", referenceType.FullName));
    }
}
```

上述函数在 ReferencePool 发生增删时调用，从中可以看出一切非抽象类均可以使用它来实现池化，但是需要派生自 IReference 强制实现 Clear 来确保回池状态被重置。

## 总结

引用池是一个优秀的设计，它通过重用对象，减少内存分配，间接地减轻 GC 压力，对性能提升有正向帮助。

另外，需要注意它和 ObjectPool 模块的区别。ReferencePool 是整个框架的基础设施，它为框架的诸多模块提供支持，其中包括 ObjectPool 模块，ObjectPool 可以说是在内部针对 ReferencePool 做了封装，然后有一些其他的功能来对业务提供帮助。简而言之，ObjectPool 更针对业务中的游戏对象，而 ReferencePool 则面向一切需要池化的类型，更加基础一些。