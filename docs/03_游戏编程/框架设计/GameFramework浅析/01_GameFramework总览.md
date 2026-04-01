## GameFramework、UnityGameFramework、StarForce

这三者都是 Ellan Jiang 大佬开源的项目（http://gameframework.cn/）

- GameFramework: 更通用的底层架构源码，它可以在任何使用 C# 的环境中进行游戏开发
- UnityGameFramework: 针对 Unity 项目对 GameFramework 进行了一层封装，它提供了基于 Component 的模块化设计以及额外功能
- StarForce: Demo，它阐述了如何使用 GF 和 UGF

## 分层

![GameFramework 分层](game_framework_01_01.png)
/// caption
GameFramework 分层
///

## 模块优先级

GF 的设计特色是模块化，资源、UI、配置、网络等等都设计了单独模块，每个模块之间存在依赖关系和弱耦合，优先级被设计用于正确处理它们之间的先后运行关系。

优先级的核心逻辑是依赖方向：被依赖的模块优先级高，这样它在 Update 时先执行，提供最新的状态给依赖它的模块。

在一个模块被构建时，按照优先级顺序被加入一个链表中，系统通过 MonoBehaviour Update 来推动这个链表中所有模块的轮循（注意这里轮循的是 GF 底层的各个 GameFrameworkModule，并非 UGF 层面的 Component），模块的轮循顺序也是按照该优先级进行。在 shutdown 时，按照优先级反序进行清理工作，确保依赖关系被正确处理。

以下是各个模块的优先级概览（默认值0）：

| 模块                | 优先级 | 说明                           |
| :------------------ | :----: | :----------------------------- |
| EventManager        | 7      | 几乎所有模块都依赖它派发事件   |
| ObjectPoolManager   | 6      | 对象池，业务和底层都对它有依赖 |
| DownloadManager     | 5      | 依赖 ObjectPool                |
| FileSystemManager   | 4      | 底层 IO                        |
| ResourceManager     | 3      | 依赖 FileSystem、ObjectPool    |
| SceneManager        | 2      | 依赖 Resource                  |
| FsmManager          | 1      | 驱动所有 FSM（含流程 FSM）     |
| ConfigManager       | 0      | 默认值                         |
| DataNodeManager     | 0      | 默认值                         |
| DataTableManager    | 0      | 默认值                         |
| EntityManager       | 0      | 默认值                         |
| LocalizationManager | 0      | 默认值                         |
| NetworkManager      | 0      | 默认值                         |
| SettingManager      | 0      | 默认值                         |
| SoundManager        | 0      | 默认值                         |
| UIManager           | 0      | 默认值                         |
| WebRequestManager   | 0      | 默认值                         |
| DebuggerManager     | -1     | 调试器，不被其他模块依赖       |
| ProcedureManager    | -2     | 依赖 FsmManager                |

## 总结

GameFramework 采用模块化架构，各个系统职责划分清晰，通过优先级推动所有模块的运行和终止。

后文开始，我按优先级逐个浅析各个系统的设计。