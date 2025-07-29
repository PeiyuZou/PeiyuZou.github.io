基础知识虽然枯燥但十分重要，反复打磨基础才能有新的启发。

List作为C#最常用的容器，有必要对它深入了解，最直接的就是读源码。下面是它的源码网址：

!!! note
    https://referencesource.microsoft.com/#mscorlib/system/collections/generic/list.cs

本篇是记录个人对它的一些常用方法源码的理解剖析

## 继承、构造、基础成员字段

```csharp
public class List<T> : IList<T>, System.Collections.IList, IReadOnlyList<T>
{
    private const int _defaultCapacity = 4;

    private T[] _items;
    private int _size;
    private int _version;

    static readonly T[]  _emptyArray = new T[0];

    // Constructs a List. The list is initially empty and has a capacity
    // of zero. Upon adding the first element to the list the capacity is
    // increased to 16, and then increased in multiples of two as required.
    public List() {
        _items = _emptyArray;
    }

    // Constructs a List with a given initial capacity. The list is
    // initially empty, but will have room for the given number of elements
    // before any reallocations are required.
    //
    public List(int capacity) {
        if (capacity < 0)
            ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.capacity, ExceptionResource.ArgumentOutOfRange_NeedNonNegNum);
        Contract.EndContractBlock();

        if (capacity == 0)
            _items = _emptyArray;
        else
            _items = new T[capacity];
    }
}
```

### 继承接口

继承IList，提供主要接口：

```csharp
#if CONTRACTS_FULL
    [ContractClass(typeof(IListContract<>))]
#endif // CONTRACTS_FULL
    public interface IList<T> : ICollection<T>
    {
        // The Item property provides methods to read and edit entries in the List.
        T this[int index] {
            get;
            set;
        }

        // Returns the index of a particular item, if it is in the list.
        // Returns -1 if the item isn't in the list.
        int IndexOf(T item);

        // Inserts value into the list at position index.
        // index must be non-negative and less than or equal to the
        // number of elements in the list.  If index equals the number
        // of items in the list, then value is appended to the end.
        void Insert(int index, T item);

        // Removes the item at position index.
        void RemoveAt(int index);
    }

#if CONTRACTS_FULL
    [ContractClassFor(typeof(IList<>))]
    internal abstract class IListContract<T> : IList<T>
    {
        T IList<T>.this[int index] {
            get {
                //Contract.Requires(index >= 0);
                //Contract.Requires(index < ((ICollection<T>)this).Count);
                return default(T);
            }
            set {
                //Contract.Requires(index >= 0);
                //Contract.Requires(index < ((ICollection<T>)this).Count);
            }
        }

        IEnumerator System.Collections.IEnumerable.GetEnumerator()
        {
            return default(IEnumerator);
        }

        IEnumerator<T> IEnumerable<T>.GetEnumerator()
        {
            return default(IEnumerator<T>);
        }

        [Pure]
        int IList<T>.IndexOf(T value)
        {
            Contract.Ensures(Contract.Result<int>() >= -1);
            Contract.Ensures(Contract.Result<int>() < ((ICollection<T>)this).Count);
            return default(int);
        }

        void IList<T>.Insert(int index, T value)
        {
            //Contract.Requires(index >= 0);
            //Contract.Requires(index <= ((ICollection<T>)this).Count);  // For inserting immediately after the end.
            //Contract.Ensures(((ICollection<T>)this).Count == Contract.OldValue(((ICollection<T>)this).Count) + 1);  // Not threadsafe
        }

        void IList<T>.RemoveAt(int index)
        {
            //Contract.Requires(index >= 0);
            //Contract.Requires(index < ((ICollection<T>)this).Count);
            //Contract.Ensures(((ICollection<T>)this).Count == Contract.OldValue(((ICollection<T>)this).Count) - 1);  // Not threadsafe
        }

        #region ICollection<T> Members

        void ICollection<T>.Add(T value)
        {
            //Contract.Ensures(((ICollection<T>)this).Count == Contract.OldValue(((ICollection<T>)this).Count) + 1);  // Not threadsafe
        }

        bool ICollection<T>.IsReadOnly {
            get { return default(bool); }
        }

        int ICollection<T>.Count {
            get {
                return default(int);
            }
        }

        void ICollection<T>.Clear()
        {
            // For fixed-sized collections like arrays, Clear will not change the Count property.
            // But we can't express that in a contract because we have no IsFixedSize property on
            // our generic collection interfaces.
        }

        bool ICollection<T>.Contains(T value)
        {
            return default(bool);
        }

        void ICollection<T>.CopyTo(T[] array, int startIndex)
        {
            //Contract.Requires(array != null);
            //Contract.Requires(startIndex >= 0);
            //Contract.Requires(startIndex + ((ICollection<T>)this).Count <= array.Length);
        }

        bool ICollection<T>.Remove(T value)
        {
            // No information if removal fails.
            return default(bool);
        }

        #endregion
    }
#endif // CONTRACTS_FULL
```

继承IReadOnlyList，则是提供泛型迭代器和Count属性等接口

```csharp
#if CONTRACTS_FULL
    [ContractClass(typeof(IReadOnlyListContract<>))]
#endif
    // If we ever implement more interfaces on IReadOnlyList, we should also update RuntimeTypeCache.PopulateInterfaces() in rttype.cs
    public interface IReadOnlyList<out T> : IReadOnlyCollection<T>
    {
        T this[int index] { get; }
    }

#if CONTRACTS_FULL
    [ContractClassFor(typeof(IReadOnlyList<>))]
    internal abstract class IReadOnlyListContract<T> : IReadOnlyList<T>
    {
        T IReadOnlyList<T>.this[int index] {
            get {
                //Contract.Requires(index >= 0);
                //Contract.Requires(index < ((ICollection<T>)this).Count);
                return default(T);
            }
        }

        int IReadOnlyCollection<T>.Count {
            get {
                return default(int);
            }
        }

        IEnumerator<T> IEnumerable<T>.GetEnumerator()
        {
            return default(IEnumerator<T>);
        }

        IEnumerator IEnumerable.GetEnumerator()
        {
            return default(IEnumerator);
        }
    }
#endif
```

### 构造函数

```csharp
// Constructs a List. The list is initially empty and has a capacity
// of zero. Upon adding the first element to the list the capacity is
// increased to 16, and then increased in multiples of two as required.
public List() {
    _items = _emptyArray;
}

// Constructs a List with a given initial capacity. The list is
// initially empty, but will have room for the given number of elements
// before any reallocations are required.
//
public List(int capacity) {
    if (capacity < 0) ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.capacity, ExceptionResource.ArgumentOutOfRange_NeedNonNegNum);
    Contract.EndContractBlock();

    if (capacity == 0)
        _items = _emptyArray;
    else
        _items = new T[capacity];
}
```

从构造函数得出，List内部也是由数组实现，并且支持在一开始就指定容量；无参构造函数的注释中，说明了数组一开始容量是0，一旦添加了一个元素，容量会增长至16，这个说法其实不对，真实是增长到4，也就是默认的容量（_defaultCapacity），后面关于Add的内容会看到算法和测试的结果；容量每次扩大都是上一次的两倍（也就是容量4再增长即是8，再扩容就是16，以此类推）。