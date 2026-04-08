GF 整个任务池的设计比较轻量化，需要注意它和传统的多线程 Task 有区别，它完全基于主线程，通过帧驱动来处理任务

## 职责划分

任务池抽象了三个角色：

- **TaskPool**: 约束任务泛型，通过帧驱动 Agent 推动任务排队和执行
- **ITaskAgent**: 任务代理，真实任务一层“壳”。它只提供了生命周期函数以及对真实任务对象的访问，用于内部接口化调度任务的生命周期，与上层逻辑解耦。
- **TaskBase**: 任务抽象类，派生 `IReference` 支持池化，它只提供了若干关键成员字段，上层派生它来实现具体的任务业务逻辑

## TaskBase

``` cs title="TaskBase"
internal abstract class TaskBase : IReference
{
    int SerialId; // 获取任务的序列编号
    string Tag; // 获取任务的标签
    int Priority; // 获取任务的优先级
    object UserData; // 获取任务的用户自定义数据
    bool Done; // 获取或设置任务是否完成
    string Description; // 获取任务描述
    void Initialize(int serialId, string tag, int priority, object userData); // 初始化
    void Clear(); // 清理
}
```

`TaskBase` 的设计足够简单，它只包含了一点点成员，这些成员在作为被管理单元时可以提供帮助。也就是任务的具体逻辑完全交给了上层实现。

## ITaskAgent 和 StartTaskStatus

``` cs title="ITaskAgent"
internal interface ITaskAgent<T> where T : TaskBase
{
    T Task; // 获取任务
    void Initialize(); // 初始化任务代理
    void Update(float elapseSeconds, float realElapseSeconds); // 任务代理轮询
    void Shutdown(); // 关闭并清理任务代理
    StartTaskStatus Start(T task); // 开始处理任务
    void Reset(); // 停止正在处理的任务并重置任务代理
}
```

任务代理除了获取任务外，只需实现几个生命周期相关的函数，其中较为关键的：

- **Update**: 任务代理轮询负责描述每帧如何更新 `TaskBase` 的业务，也可能在这里将 `TaskBase.Done` 标记为 `true`，以此来表示该任务完成
- **Start**: 开始处理任务，这个函数返回一个状态枚举，它用于任务池判断如何继续处理该任务

``` cs title="StartTaskStatus"
public enum StartTaskStatus : byte
{
    /// <summary>
    /// 可以立刻处理完成此任务。
    /// </summary>
    Done = 0,

    /// <summary>
    /// 可以继续处理此任务。
    /// </summary>
    CanResume,

    /// <summary>
    /// 不能继续处理此任务，需等待其它任务执行完成。
    /// </summary>
    HasToWait,

    /// <summary>
    /// 不能继续处理此任务，出现未知错误。
    /// </summary>
    UnknownError
}
```

需要注意的是，该状态并不代表任务的状态，而是“任务开始”这一瞬间的状态。也就是说:

- **`Done`**: 代表这一帧已经处理完了任务，任务的全部处理逻辑在 Start 函数中已经完成，不需要靠帧推动继续执行
- **`CanResume`**: 代表任务成功跑起来了，可以在 Update 中继续推动更新
- **`HasToWait`**: 代表任务因为某些原因暂时无法开始，需要继续等待直到可以开始执行
- **`UnknownError`**: 代表任务出错无法执行

## TaskPool

`TaskPool` 的成员我们只看关键的几个：

``` cs title="TaskPool"
internal sealed class TaskPool<T> where T : TaskBase
{
    Stack<ITaskAgent<T>> m_FreeAgents; // 空闲代理栈
    GameFrameworkLinkedList<ITaskAgent<T>> m_WorkingAgents; // 工作中的代理链表
    GameFrameworkLinkedList<T> m_WaitingTasks; // 等待处理的任务

    public void Update(float elapseSeconds, float realElapseSeconds); // 任务池轮询
    public void AddAgent(ITaskAgent<T> agent); // 增加任务代理
    public void AddTask(T task); // 增加任务
    public bool RemoveTask(int serialId); // 根据任务的序列编号移除任务
    public int RemoveTasks(string tag); // 根据任务的标签移除任务
    public int RemoveAllTasks(); // 移除所有任务
    private void ProcessRunningTasks(float elapseSeconds, float realElapseSeconds); // 处理运行中的任务
    private void ProcessWaitingTasks(float elapseSeconds, float realElapseSeconds); // 处理等待中的任务
}
```

任务池的运作完全围绕这三个核心数据结构进行，通过代理来一对一处理任务，代理不够则剩下的任务继续等待。

代理完全通过 `AddAgent` 添加，如果需要 10 个代理工作，外部需要添加 10 个。

而任务的添加和移除则是通过 `AddTask`、`RemoveTask`、`RemoveTasks`、`RemoveAllTasks` 来完成，添加任务只需要加入等待链表即可，而移除稍微复杂一些，如果这个任务已经开始运行起来，则还需要将工作代理解放出来。

整个池子的运作逻辑核心在 `ProcessRunningTasks` 和 `ProcessWaitingTasks` 两个函数，它们被 Update 驱动：

- **`ProcessRunningTasks`** 每帧推动任务代理更新，以此间接推动任务的更新，当发现有任务完成时，将任务结束回收，并解放它的代理
- **`ProcessWaitingTasks`** 在发现有闲置的代理时，将等待中的任务排队拿出来，然后尝试让代理开始任务，代理开始任务会返回一个状态，也就是前面提到的 `StartTaskStatus`，只有当任务状态是 `CanResume` 时，才会将该代理分配在 `m_WorkingAgents` 中，以便 Update 继续推动处理该任务。

## TaskInfo 和 TaskStatus

这两不赘述，用于调试和查看任务对象信息，可以通过 `TaskPool` 获取当前池内的信息。

## 总结

GameFramework的任务池是一个基于优先级 + 帧驱动 + 限流执行 + 对象复用的任务调度器，简单易用，但需要注意它只支持在主线程上处理任务