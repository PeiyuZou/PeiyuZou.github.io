## 1. 模块概览

ProcedureManager（流程管理器）是 GameFramework 中用于管理游戏整体流程（如启动流程、登录流程、主界面流程、游戏中流程等）的模块。它本质上是对 **FSM（有限状态机）的一层业务封装**，每个"流程"就是状态机中的一个"状态"。

---

## 2. 类型结构

```
GameFrameworkModule          ← 所有模块的抽象基类
    └── ProcedureManager     ← 流程管理器（internal sealed）

FsmState<T>                  ← FSM 状态基类（泛型）
    └── ProcedureBase        ← 流程基类（public abstract），T = IProcedureManager
            └── 你的具体流程（如 ProcedureLaunch、ProcedureMain...）
```

`ProcedureManager` 实现了 `IProcedureManager` 接口，外部代码只通过接口访问，不依赖具体实现。

---

## 3. 核心设计：委托给 FsmManager

ProcedureManager **自身的 `Update` 是空的**：

```csharp
// ProcedureManager.cs:79
internal override void Update(float elapseSeconds, float realElapseSeconds)
{
    // 什么都不做
}
```

流程的实际驱动来自 `Initialize` 时将 FSM 注册到 `FsmManager`：

```csharp
// ProcedureManager.cs:105
public void Initialize(IFsmManager fsmManager, params ProcedureBase[] procedures)
{
    m_FsmManager = fsmManager;
    m_ProcedureFsm = m_FsmManager.CreateFsm(this, procedures);
    //                             ↑ Owner 是 IProcedureManager 自身
}
```

之后每帧由 `FsmManager.Update` 负责驱动所有 FSM（包括 `m_ProcedureFsm`）的 tick：

```csharp
// FsmManager.cs:58
internal override void Update(float elapseSeconds, float realElapseSeconds)
{
    foreach (FsmBase fsm in m_TempFsms)
    {
        fsm.Update(elapseSeconds, realElapseSeconds); // 驱动 ProcedureFsm
    }
}
```

**结论：ProcedureManager 是一个"壳"，真正的 tick 链路是：**

```
GameFrameworkEntry.Update()
    → FsmManager.Update()           (Priority=1)
        → Fsm<IProcedureManager>.Update()
            → CurrentProcedure.OnUpdate()   ← 你的游戏逻辑写在这里
```

---

## 4. 模块优先级系统

### 4.1 优先级的作用

在 `GameFrameworkEntry` 中，所有模块按 Priority **从高到低**插入一条链表：

```csharp
// GameFrameworkEntry.cs:110
LinkedListNode<GameFrameworkModule> current = s_GameFrameworkModules.First;
while (current != null)
{
    if (module.Priority > current.Value.Priority)
        break;
    current = current.Next;
}
// 插入到第一个比自己优先级低的模块之前
```

**Update（轮询）**：正向遍历链表 → Priority 高的先执行。
**Shutdown（关闭）**：反向遍历链表 → Priority 高的后关闭。

### 4.2 各模块优先级一览

| 模块 | Priority | 说明 |
|---|:---:|---|
| EventManager | 7 | 最高，几乎所有模块都靠它派发事件 |
| ObjectPoolManager | 6 | 对象池，Resource/Entity 等强依赖 |
| DownloadManager | 5 | 依赖 ObjectPool |
| FileSystemManager | 4 | 底层 IO，Resource 依赖 |
| ResourceManager | 3 | 依赖 FileSystem、ObjectPool |
| SceneManager | 2 | 依赖 Resource |
| FsmManager | 1 | 驱动所有 FSM（含流程 FSM） |
| 其他未重写的模块 | 0 | 基类默认值 |
| DebuggerManager | -1 | 调试器，不被其他模块依赖 |
| **ProcedureManager** | **-2** | 最低，自身 Update 为空，依赖 FsmManager |

### 4.3 ProcedureManager 为什么优先级最低

- **Update 方面**：ProcedureManager 的 Update 是空实现，放在最后执行没有任何影响，因为驱动流程的工作已经由 FsmManager（Priority=1）完成了。
- **Shutdown 方面**：优先级最低意味着它**最先被 Shutdown**。流程（Procedure）是最高层的业务逻辑，它依赖所有底层模块。因此在关闭时必须最先停止流程，再逐步释放它所依赖的底层资源。

```
Shutdown 顺序（Priority 低 → 高）：
ProcedureManager(-2) → DebuggerManager(-1) → ... → FsmManager(1) → EventManager(7)
```

---

## 5. 生命周期

```
①  GameFrameworkEntry.GetModule<IProcedureManager>()
        → 自动创建 ProcedureManager 并按 Priority 插入链表

②  procedureManager.Initialize(fsmManager, procedures)
        → 在 FsmManager 中创建 Fsm<IProcedureManager>
        → 所有 ProcedureBase 以 FsmState 身份注册进 FSM，触发 OnInit

③  procedureManager.StartProcedure<T>()
        → 调用 m_ProcedureFsm.Start<T>()
        → 触发首个流程的 OnEnter

④  每帧 GameFrameworkEntry.Update()
        → FsmManager.Update() 驱动 FSM tick
        → 当前流程 OnUpdate() 执行
        → 流程内调用 ChangeState<T>() 触发 OnLeave / OnEnter 切换流程

⑤  GameFrameworkEntry.Shutdown()
        → 反向遍历，ProcedureManager 最先 Shutdown
        → 销毁 m_ProcedureFsm（触发当前流程 OnLeave + 所有流程 OnDestroy）
        → FsmManager 随后 Shutdown，清理所有 FSM
```

---

## 6. 流程切换

流程切换通过 FSM 的状态切换实现，在具体的 `ProcedureBase` 子类中调用：

```csharp
// 在某个 ProcedureBase 子类中
protected internal override void OnUpdate(ProcedureOwner procedureOwner, ...)
{
    if (加载完成)
    {
        ChangeState<ProcedureMain>(procedureOwner); // 切换到主流程
    }
}
```

`ProcedureOwner` 是 `IFsm<IProcedureManager>` 的别名（typedef），提供状态切换、数据存取等能力。

---

## 7. 设计总结

| 设计点 | 具体体现 |
|---|---|
| **职责单一** | ProcedureManager 只负责管理流程集合和提供查询接口，不自己驱动 tick |
| **依赖注入** | FsmManager 通过 Initialize 参数传入，而非硬编码获取 |
| **接口隔离** | 外部通过 `IProcedureManager` 访问，内部实现对外不可见（`internal sealed`） |
| **优先级即依赖序** | Priority 直接反映模块间的依赖方向，是系统启动/关闭顺序的唯一依据 |
| **流程即状态** | `ProcedureBase : FsmState<IProcedureManager>`，复用 FSM 完整的状态生命周期 |
