## 一、原理

LoadFromFile 是目前移动平台最常见的加载方式，它的优势核心在于它的加载原理。

在加载 AB 包时并不会将整个文件读入内存，而是仅仅加载主文件中的文件头和一些元数据（BlockInfo、TypeTree、DirectoryInfo等），在需要加载其中的资产时：

- **LZMA格式的包：**解压整包并加载需要的数据
- **LZ4格式的包：**根据元数据找到并解压对应的若干 Chunk 数据块，然后从中加载资产数据

由于我们通常都使用 LZ4 的包，所以 LoadFromFile 的机制让它的整体内存峰值表现很好，并且加载速度很快。

!!! warning "常见误解"
    很多人说 LoadFromFile 是零额外内存开销，但实际上不是，如果你需要从中加载资产数据，那么对应 Chunk 的磁盘数据肯定要被加载进内存，然后解压出真正的数据（此时仍然是文件内容布局），最后再序列化为资产对象。并不是直接从磁盘数据直接序列化得到资产对象，所以内存上还是一样的有额外分配。

## 二、优点

- 性能最优，加载速度最快
- 内存占用最小
- 支持所有压缩格式

## 三、缺点

- 基本没有

## 四、代码示例

注意以下示例均为测试的写法法，商业化的做法中，上层只需要传入资产的相对路径即可，不管是 AB 的路径还是资产名字都是根据相对路径自动计算得到的。

### 1. LoadFromFile

```cs
void LoadPrefab(string abRelativePath, string prefabName)
{
    // 构建AB文件路径
    string path = Path.Combine(
        Application.streamingAssetsPath,
        abRelativePath
    );

    // 同步加载AB
    AssetBundle ab = AssetBundle.LoadFromFile(path);

    if (ab != null)
    {
        // 从AB中加载资源
        GameObject prefab = ab.LoadAsset<GameObject>(prefabName);

        if (prefab != null)
        {
            // 实例化对象
            GameObject instance = Instantiate(prefab);
            Debug.Log($"成功加载并实例化: {prefab.name}");
        }

        // 注意：不要立即卸载AB，否则prefab会丢失引用
        // ab.Unload(false); // ❌ 错误！
    }
    else
    {
        Debug.LogError($"AB加载失败: {path}");
    }
}
```

### 2. LoadFromFileAysnc

```cs
IEnumerator LoadPrefabAsync(string abRelativePath, string prefabName)
{
    string path = Path.Combine(
        Application.streamingAssetsPath,
        abRelativePath
    );

    Debug.Log("开始异步加载AB...");

    // 开始异步加载
    AssetBundleCreateRequest request = AssetBundle.LoadFromFileAsync(path);

    // 等待加载完成，可以显示进度
    while (!request.isDone)
    {
        float progress = request.progress * 100f;
        Debug.Log($"加载进度: {progress:F1}%");
        yield return null;
    }

    // 获取加载的AB
    AssetBundle ab = request.assetBundle;

    if (ab != null)
    {
        Debug.Log("AB加载完成，开始加载资源...");

        // 异步加载资源
        AssetBundleRequest assetRequest =
            ab.LoadAssetAsync<GameObject>(prefabName);

        yield return assetRequest;

        GameObject prefab = assetRequest.asset as GameObject;

        if (prefab != null)
        {
            Instantiate(prefab);
            Debug.Log("资源加载并实例化成功!");
        }
    }
    else
    {
        Debug.LogError("AB加载失败!");
    }
}
```