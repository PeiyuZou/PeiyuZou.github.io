## 一、原理

```cs title="核心语句"
UnityWebRequest request = UnityWebRequestAssetBundle.GetAssetBundle(url);
```

Unity 专为 AB 设计的网络加载 API，特点：

- 自动缓存管理
- 支持HTTPS
- 断点续传
- 基于Hash的版本控制

## 二、优点

- 自动缓存管理
- 支持断点续传
- 支持HTTPS安全传输
- 基于Hash的精确版本控制
- 支持进度显示
- Unity官方推荐

## 三、缺点

- 只能用于HTTP/HTTPS
- 首次下载需要网络
- 需要服务器支持

## 四、代码示例

```cs title="基本用法"
IEnumerator DownloadAB()
{
    string url = "https://yourserver.com/assetbundles/myassetbundle";

    // 创建下载请求
    UnityWebRequest request =
        UnityWebRequestAssetBundle.GetAssetBundle(url);

    // 发送请求
    yield return request.SendWebRequest();

    // 检查结果
    if (request.result == UnityWebRequest.Result.Success)
    {
        // 获取AB
        AssetBundle ab = DownloadHandlerAssetBundle.GetContent(request);

        if (ab != null)
        {
            GameObject prefab = ab.LoadAsset<GameObject>("MyPrefab");
            Instantiate(prefab);

            Debug.Log("AB下载并加载成功!");
        }
    }
    else
    {
        Debug.LogError($"下载失败: {request.error}");
    }

    // 释放请求资源
    request.Dispose();
}
```

## 五、缓存系统

### 使用版本号缓存

```csharp
IEnumerator DownloadWithVersion()
{
    string url = "https://yourserver.com/assetbundles/myassetbundle";
    uint version = 2;  // AB版本号

    // 使用版本号，Unity会自动管理缓存
    UnityWebRequest request =
        UnityWebRequestAssetBundle.GetAssetBundle(url, version, 0);

    yield return request.SendWebRequest();

    if (request.result == UnityWebRequest.Result.Success)
    {
        AssetBundle ab = DownloadHandlerAssetBundle.GetContent(request);
        Debug.Log("AB加载成功（可能来自缓存）");
    }

    request.Dispose();
}
```

### 使用Hash128缓存（推荐）

```csharp
IEnumerator DownloadWithHash()
{
    string url = "https://yourserver.com/assetbundles/myassetbundle";

    // 从Manifest获取Hash
    Hash128 hash = manifest.GetAssetBundleHash("myassetbundle");

    // 或者手动指定Hash
    // Hash128 hash = Hash128.Parse("d4735e3a265e16eee03f59718b9b5d03");

    // 使用Hash版本控制（推荐！）
    UnityWebRequest request =
        UnityWebRequestAssetBundle.GetAssetBundle(url, hash, 0);

    yield return request.SendWebRequest();

    if (request.result == UnityWebRequest.Result.Success)
    {
        AssetBundle ab = DownloadHandlerAssetBundle.GetContent(request);
        Debug.Log("AB加载成功");
    }

    request.Dispose();
}
```

### 缓存管理工具类

```csharp
using UnityEngine;
using UnityEngine.Networking;

public static class ABCacheManager
{
    /// <summary>
    /// 检查AB是否已缓存
    /// </summary>
    public static bool IsCached(string url, Hash128 hash)
    {
        return Caching.IsVersionCached(url, hash);
    }

    /// <summary>
    /// 清除所有缓存
    /// </summary>
    public static void ClearAllCache()
    {
        Caching.ClearCache();
        Debug.Log("所有AB缓存已清除");
    }

    /// <summary>
    /// 清除特定版本的缓存
    /// </summary>
    public static void ClearCachedVersion(string url, Hash128 hash)
    {
        if (Caching.IsVersionCached(url, hash))
        {
            Caching.ClearCachedVersion(url, hash);
            Debug.Log($"已清除缓存: {url}");
        }
    }

    /// <summary>
    /// 获取缓存信息
    /// </summary>
    public static void PrintCacheInfo()
    {
        long cacheSize = Caching.defaultCache.spaceOccupied;
        Debug.Log($"缓存占用空间: {cacheSize / 1024 / 1024}MB");
        Debug.Log($"缓存路径: {Caching.defaultCache.path}");
    }

    /// <summary>
    /// 设置缓存参数
    /// </summary>
    public static void ConfigureCache()
    {
        // 启用压缩
        Caching.compressionEnabled = true;

        // 设置最大缓存空间 (1GB)
        Caching.maximumAvailableDiskSpace = 1024L * 1024 * 1024;

        Debug.Log("缓存配置完成");
    }
}
```

### 缓存位置

不同平台的缓存路径：

| 平台 | 缓存路径 |
|------|---------|
| Windows | `%LocalAppData%\Unity\[CompanyName]\[ProductName]\Cache` |
| Mac | `~/Library/Caches/Unity/[CompanyName]/[ProductName]` |
| Android | `Application.persistentDataPath/Unity/Cache` |
| iOS | `Application.persistentDataPath/Unity/Cache` |

## 六、错误处理

```csharp
IEnumerator DownloadWithErrorHandling()
{
    string url = "https://yourserver.com/assetbundles/myassetbundle";
    int maxRetries = 3;
    int currentRetry = 0;

    while (currentRetry < maxRetries)
    {
        UnityWebRequest request =
            UnityWebRequestAssetBundle.GetAssetBundle(url);

        yield return request.SendWebRequest();

        // 检查结果
        switch (request.result)
        {
            case UnityWebRequest.Result.Success:
                // 成功
                AssetBundle ab = DownloadHandlerAssetBundle.GetContent(request);
                Debug.Log("下载成功!");
                request.Dispose();
                yield break;  // 退出重试循环

            case UnityWebRequest.Result.ConnectionError:
                // 网络连接错误
                Debug.LogWarning($"网络错误: {request.error}");
                break;

            case UnityWebRequest.Result.ProtocolError:
                // HTTP错误（404, 500等）
                Debug.LogWarning($"HTTP错误 {request.responseCode}: {request.error}");
                break;

            case UnityWebRequest.Result.DataProcessingError:
                // 数据处理错误
                Debug.LogWarning($"数据错误: {request.error}");
                break;
        }

        request.Dispose();
        currentRetry++;

        if (currentRetry < maxRetries)
        {
            Debug.Log($"重试 {currentRetry}/{maxRetries}...");
            yield return new WaitForSeconds(2f);  // 等待2秒后重试
        }
    }

    Debug.LogError("下载失败，已达到最大重试次数");
}
```