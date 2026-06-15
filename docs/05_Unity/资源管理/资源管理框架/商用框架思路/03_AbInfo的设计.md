## AbInfo 做什么？

AbInfo 本质上是对单个 AB 包的封装，所以它最核心的职责其实是：

1. 获取或加载其所属的 AB 包
2. 从包中获取或加载资源
3. 执行完成回调

但实际业务情况会导致实际职责远比这个复杂，还需要考虑以下问题的处理：

- 对外有 4 种加载请求：纯 AB / 单 Asset / 全 Asset / Sprite。每种都可能被并发调用 N 次
- 对内 N 个依赖 AB 同时在加载
- 每个请求都可能是同步或异步，异步途中可能被一个同步请求 "插队"
- 回调链非常长：依赖加载完 → 自身 AB 加载完 → Asset 加载完 → 触发上层回调 → 引用清零 → 进回收站

## 一、五个加载状态

```cs title="五个加载状态"
public enum LoadState
{
    Unloaded,  // 未开始加载
    Waiting,   // 在等待列表中（资源等待加载）
    Loading,   // 加载中
    Loaded,    // 已加载
    Dispatch,  // 回调中，防止重复触发回调
}
```

关键的是 Dispatch 这个状态——很多团队的状态机里没有这一档，但这里是必须的：

```cs title="回调里面可能再次调用Load，形成重入"
UI.OnIconLoaded(sprite) {
    // 拿到 icon 后立即加载同一张图的另一种变体
    ResMgr.LoadSprite("ui/icon_xxx_dark", OnDarkLoaded);
}
```

用 Dispatch 临时占位，让嵌套调用知道 "我现在正在派发，先排队等下一波"。

## 二、同步加载与异步加载

控制使用同步还是异步是由 AbSync 开关决定的，但它设计为单向的，只允许异步转同步。

```cs title="AbSync是单向开关"
private bool AbSync
{
    get { return mIsSync; }
    set { mIsSync |= value; }   // ← 注意是 |=，不是 =
}
```

一旦某次请求要求"同步加载"，整个 AB 的加载就必须升级成同步，因为同步的本质是 "调用返回前必须拿到资源"。所以这是个 "涨潮模式"，只能涨不能退。

```cs title="异步转同步"
private void LoadAb()
{
    switch (AbState)
    {
        case LoadState.Unloaded:
            if (AbSync) LoadAbSync();
            else        mAbManager.LoadAbAsync(this);
            break;
        case LoadState.Waiting:
            if (AbSync) LoadAbSync();   // ← 异步等待中，强行转同步
            break;
        case LoadState.Loading:
            break;                       // 已经在异步加载了，回调里会触发
        case LoadState.Loaded:
            TestAllAbsLoaded();
            break;
        ...
    }
}
```

"异步转同步"的实现很优雅：直接在原地调一次 LoadAbSync()。同步调用一返回，整个状态机就推进到 Loaded，之前异步链路的 Update 轮询发现 "AB 已经加载好了" 就自然停止。

LoadState.Loading 分支什么都不做，因为底层 AssetBundle.LoadFromFileAsync 已经在跑，没法取消。这意味着 "加载已经在进行中的资源" 做不了同步，只能等异步完成。这是 Unity API 的物理限制。

## 三、完整加载时序图

```cs title="完整加载时序图"
LoadAsset("xxx", cb, sync=false)
   │
   ├─ AddCallback("xxx", cb)
   ├─ AbSync |= false
   ├─ InitDependency(sync=false)
   │     ├─ 找出 depA、depB、depC
   │     ├─ depA.LoadAb()  (异步)  ──┐
   │     ├─ depB.LoadAb()  (异步)    │
   │     ├─ depC.LoadAb()  (异步)    │
   │     └─ AddDependency × 3        │
   │            ├─ A 已 Loaded → 直接 AddRef
   │            └─ 其它 → WaitForMe + ++unloadedDependCount
   └─ LoadAb()  (异步)  ── 进入 AbManager.LoadAbAsync 队列
                                              │
   Update 轮询：                              │
     依赖们陆续完成 → 各自触发 OnAbLoaded     │
        ├─ foreach (waitForMe) → OnDependLoaded
        │       └─ --unloadedDependCount
        │       └─ TestAllAbsLoaded()  // 如果都好了，就……
        ▼
   本 AB 也完成 → OnAbLoaded
        └─ TestAllAbsLoaded()
                ├─ 触发 mAbCbList 全部回调
                ├─ 有 LoadAll 请求 → LoadAll()
                ├─ 有 Sprite 请求 → LoadAllSprites()
                └─ 有 Asset 请求 → InnerLoadAsset() × N
                          │
                          ▼
                   Asset 加载完成 → InvokeCallbacks(assetInfo)
                          │
                          └─ TryRelease()  // 计数 0 → 进回收站
```