## 一、原理

LoadFromMemory 本质上是从一个指定的 byte 数组中创建一个 AB 包对象。一直以来它被人诟病的是双倍内存峰值问题：

- 第一份：byte[]数组（压缩的AB数据）
- 第二份：解压后的AB对象

``` title="双倍内存峰值"
  byte数组: 10MB (压缩数据)
  AB对象: 30MB (解压后)
  总计: 40MB ❌
  峰值: 立即占用40MB
```

但其实第一份并不算是 LoadFromMemory 自身造成的，而是由于它需要传入一个字节序列，所以外部会提前将 AB 以字节数组的形式加载进内存；它本身只造成第二份内存的分配。

但总之，最后的结果是双倍内存的占用，它相比于 LoadFromFile 没有什么优势，所以现在使用很少很少，除非有某些特殊情况，最后导致 AB 不得不以字节数组的形式在内存中分配。

## 二、优点

- 可以从任意来源获取数据（网络、加密文件等）
- 不需要文件系统支持

## 三、缺点

- **双倍内存峰值！**
- 加载速度慢（需要完整解压）

## 四、代码示例

### 1. LoadFromMemory

```cs
void LoadABFromMemory()
{
    string path = Path.Combine(
        Application.streamingAssetsPath,
        "myassetbundle"
    );

    // 1. 读取整个文件到内存（第一份内存）
    byte[] fileData = File.ReadAllBytes(path);
    Debug.Log($"文件大小: {fileData.Length / 1024}KB");

    // 2. 从内存创建AB（第二份内存）
    AssetBundle ab = AssetBundle.LoadFromMemory(fileData);

    // ⚠️ 此时内存中有两份数据：
    // - fileData 数组（压缩的）
    // - AB对象（解压的）

    if (ab != null)
    {
        GameObject prefab = ab.LoadAsset<GameObject>("MyPrefab");
        Instantiate(prefab);

        Debug.Log("从内存加载AB成功");
    }

    // fileData 依然占用内存，直到GC回收
}
```

### 2. LoadFromMemoryAsync

```cs
IEnumerator LoadABAsync()
{
    string path = Path.Combine(
        Application.streamingAssetsPath,
        "myassetbundle"
    );

    // 读取文件到内存（这步仍然是同步的！）
    byte[] fileData = File.ReadAllBytes(path);
    Debug.Log($"文件读取完成: {fileData.Length / 1024}KB");

    // 异步从内存创建AB（解压过程是异步的）
    AssetBundleCreateRequest request =
        AssetBundle.LoadFromMemoryAsync(fileData);

    while (!request.isDone)
    {
        Debug.Log($"解压进度: {request.progress * 100:F1}%");
        yield return null;
    }

    AssetBundle ab = request.assetBundle;

    if (ab != null)
    {
        GameObject prefab = ab.LoadAsset<GameObject>("MyPrefab");
        Instantiate(prefab);
    }
}
```

!!! warning "注意事项"
    虽然LoadFromMemoryAsync是异步的，但 `File.ReadAllBytes()` 仍然是同步的，会阻塞主线程

!!! note "关于LoadFromMemory的解密"
    由于 LoadFromMemory 是直接从整个字节序列中创建 AB，并不是流模式，所以解密其实和它本身无关，一般是 File.ReadAllBytes 先将加密后的 AB 文件以字节序列的方式读起来，然后对这个字节序列解密得到可用的字节序列