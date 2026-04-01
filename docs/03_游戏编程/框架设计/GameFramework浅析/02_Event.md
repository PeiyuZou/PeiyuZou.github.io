## 1. 接口设计

从接口设计入手，可以知道一个系统大致的行为，对外提供什么样的服务。

![IEventManager](gf_02_01.png)
/// caption
IEventManager
///

关键的几个函数：

- **Check**: 检查是否存在指定事件处理函数
- **Subscribe**: 订阅事件
- **Unsubscribe**: 取消订阅事件
- **Fire**: 线程安全的抛出事件，事件会在抛出后的下一帧分发
- **FireNow**: 抛出事件立即模式，这个操作不是线程安全的，事件会立刻分发

## 2. 内部实现

EventManager 只是对单个 EventPool 的封装，它构建一个指定模式的 EventPool :

- **Default**: 默认事件池模式，即必须存在有且只有一个事件处理函数
- **AllowNoHandler**: 允许不存在事件处理函数
- **AllowMultiHandler**: 允许存在多个事件处理函数
- **AllowDuplicateHandler**: 允许存在重复的事件处理函数

EventManager 默认构建了一个模式为 `EventPoolMode.AllowNoHandler | EventPoolMode.AllowMultiHandler` 的池，也许是大部分底层或者业务逻辑用这种模式都兼容。

核心的实现在 EventPool 中：

### 2.1 数据结构

- **m_EventHandlers**: 存所有事件Id和响应函数的映射关系，这里用了键值一对多的字典，因为单个事件Id可能同时对应复数个响应函数，字典内部复数的值使用一个尾插法链表维护
- **m_Events**: 线程安全的事件队列，由 Update 推动出队消耗，仅处理 Fire 函数派发的事件
- **m_CachedNodes**: 派发时的游标缓存
- **m_TempNodes**: 取消订阅时的临时缓冲
- **m_DefaultHandler**: 兜底处理函数

### 2.2 核心数据结构：GameFrameworkMultiDictionary

这是整个 EventPool 的基础。它用**一条共享链表 + 一个字典**来存储「一个 Key → 多个 Value」的映射。

![GameFrameworkMultiDictionary](gf_02_02.png)
/// caption
GameFrameworkMultiDictionary
///

- **Terminal 节点**：每个 key 对应一个哨兵节点（值为 null），标记该 key 的 handler 列表结束位置
- **Range**：不是一段连续内存，而是链表上的 `[First, Terminal)` 半开区间
- **遍历规则**：`current != null && current != range.Terminal`，到哨兵就停

### 2.3 订阅事件

订阅事件逻辑比较简单，核心就是在 `m_EventHandlers` 对应的桶链表中，将当前 `handler` 尾插进去（在 Terminal 节点前）

``` title="订阅事件尾插示意"
链表变化：
  Before: [HandlerA1] → [TerminalA]
  After:  [HandlerA1] → [HandlerA2(新)] → [TerminalA]
                              ↑
                          插入位置
```

### 2.4 取消订阅事件

取消订阅稍微复杂一些，这里使用了两个额外的缓存区来，主要是为了解决**重入调用**问题：

``` title="重入调用示例"
// 主线程执行
EventPool.FireNow(this, event)
HandleEvent 开始遍历 event 的 handlers
    1. 调用 handlerA
        1.1 handlerA 内部调用 Unsubscribe(event.Id, handlerB) // 关键！！！
    2. 继续调用 handlerB // 已被移除！！！
    3. 等待调用 handlerC // 已打断无法触达
```

所以 `HandleEvent` 在遍历 handlers 链表时，执行完 `current` 之后并没有直接通过 `current = current.Next` 取下一个节点，而是从 `m_CachedNodes` 中获取

`m_CachedNodes` 会提前获取 `current.Next`，如果没有发生重入调用，那么取出来自然就是期望的下一节点；如果发生了重入调用，取消订阅时会将 `m_CachedNodes` 缓存的下一节点变为再下一个节点，在上面的例子中就是 `handlerC`，下面是关键代码：

``` cs title="遍历链表的处理"
if (m_EventHandlers.TryGetValue(e.Id, out range))
{
    LinkedListNode<EventHandler<T>> current = range.First;
    while (current != null && current != range.Terminal)
    {
        // 提前缓存下一个节点
        m_CachedNodes[e] =
            current.Next != range.Terminal ? current.Next : null;
        // 执行当前节点对应的 handler，这一步有可能发生重入调用
        current.Value(sender, e);
        // 从缓存中拿正确节点，缓存自动处理节点失效
        current = m_CachedNodes[e];
    }
    // 使用完毕丢掉
    m_CachedNodes.Remove(e);
}
```

``` cs title="取消订阅时缓存的自动处理"
if (m_CachedNodes.Count > 0)
{
    foreach (KeyValuePair<object, LinkedListNode<EventHandler<T>>> cachedNode in m_CachedNodes)
    {
        // 发生了重入：当前取订的目标 = 下一个执行节点
        if (cachedNode.Value != null && cachedNode.Value.Value == handler)
        {
            // 发生重入的处理：取下下个节点
            m_TempNodes.Add(cachedNode.Key, cachedNode.Value.Next);
        }
    }

    if (m_TempNodes.Count > 0)
    {
        foreach (KeyValuePair<object, LinkedListNode<EventHandler<T>>> cachedNode in m_TempNodes)
        {
            // 将取订目标自动更新为下下个节点
            // 现在从缓存中取出来就保证正常了
            // old: A->B  new: A->C
            m_CachedNodes[cachedNode.Key] = cachedNode.Value;
        }

        m_TempNodes.Clear();
    }
}

// 取消订阅事件
if (!m_EventHandlers.Remove(id, handler))
{
    throw new GameFrameworkException(Utility.Text.Format("Event '{0}' not exists specified handler.", id));
}
```

### 2.5 派发事件

GF 设计的派发分为两种：**立即派发（FireNow）**、**线程安全的事件队列（Fire）**

FireNow 的逻辑很简单，只是遍历 handlers 链表，挨个调用。

``` cs title="立即派发"
/// <summary>
/// 抛出事件立即模式，这个操作不是线程安全的，事件会立刻分发。
/// </summary>
/// <param name="sender">事件源。</param>
/// <param name="e">事件参数。</param>
public void FireNow(object sender, T e)
{
    if (e == null)
    {
        throw new GameFrameworkException("Event is invalid.");
    }

    HandleEvent(sender, e);
}
```

Fire 则是为了确保线程安全，加锁将事件调用推进队列中，Update 在下一帧会出队消耗完成调用

``` cs title="线程安全的事件队列"
/// <summary>
/// 抛出事件，这个操作是线程安全的，即使不在主线程中抛出，
/// 也可保证在主线程中回调事件处理函数，但事件会在抛出后的下一帧分发。
/// </summary>
/// <param name="sender">事件源。</param>
/// <param name="e">事件参数。</param>
public void Fire(object sender, T e)
{
    if (e == null)
    {
        throw new GameFrameworkException("Event is invalid.");
    }

    Event eventNode = Event.Create(sender, e);
    lock (m_Events)
    {
        m_Events.Enqueue(eventNode);
    }
}
```

## 3.总结

GameFramework 的事件系统除了实现基本的发布订阅模式外，还考虑了线程安全和重入调用的问题。使用时需要注意 Fire 和 FireNow 的区别，另外重入调用的解决方案也值得一看。