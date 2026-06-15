## 一、Resource模式

Resources 模式目前已经被市面大多项目弃用了，因为它有一些很不方便的地方：

- 凡是在 Resources 文件夹中的资产会自动进包，做分包和打空包不方便
- 对打真机包流程来说不方便，一般要再建一个镜像工程，只包含 AB 且不包含 Resources

使用它的唯一意义可能就是对小型项目友好，不打 AB 直接出包跑真机的情况，但一般不管是什么规模的项目，都推荐使用 AB ，就算是 Addressable 本质也是在使用 AB 。

它的整体加载设计和 AB 大同小异，直接跳过。

## 二、NetworkTextureManager：独立的远程图片子系统

### 1. 两个静态方法

```cs title="外部使用的静态方法"
NetworkTextureManager.LoadRemoteTexture(url, w, h, matchWH, OnLoaded);
NetworkTextureManager.UnLoadRemoteTexture(shortName, OnLoaded);
```

### 2. 三个状态机

```cs title="waiting / loading / loaded"
private static LinkedList<TextureCache>         waiting = new();
private static Dictionary<string, TextureCache> loading = new();
private static Dictionary<string, TextureCache> loaded  = new();
```

### 3. 请求合并

```cs title="合并"
// 1. loaded：直接复用
if (loaded.TryGetValue(shortName, out cache))
{
    cache.AddRef();
    callback(cache.Texture);
    return shortName;
}

// 2. loading：合并到 Callbacks 链
if (loading.TryGetValue(shortName, out cache))
{
    cache.Callbacks += callback;
    cache.AddRef();
    return shortName;
}

// 3. waiting：同样合并
if (waiting.Count > 0) { 遍历找同名项，合并; }

// 4. 都不在 → 新建，进 waiting
cache = Spwan(shortName);
cache.Callbacks = callback;
waiting.AddLast(Spwan(cache));
DispatchTask();
```

远程图片下载慢且 UI 上经常同一张头像被多个组件用（聊天框、邮件列表、好友列表）。每张图都重新下，浪费带宽也浪费时间。用 shortName（URL hash）做 key，相同 URL 的请求合并成一次实际下载，全部回调一次性触发。

### 4. 三层缓存：内存 / 本地文件 / 远程

``` title="三层缓存"
LoadRemoteTexture(url)
   │
   ├─ loaded 字典命中？  → AddRef + 立即回调
   ├─ loading 字典命中？ → Callbacks 合并
   ├─ waiting 队列命中？ → Callbacks 合并
   │
   └─ 加入 waiting，等待 DispatchTask 派发
              │
              ▼
     LoadTextureFromFile(shortName)     ← 本地磁盘缓存
              │
              ├─ 命中 → 直接生成 Texture2D，进 loaded
              │
              └─ 未命中 → DoLoadRemoteTexture (UnityWebRequest)
                          │
                          └─ 下载成功 → CompressTexture → 写本地文件 → 进 loaded
```