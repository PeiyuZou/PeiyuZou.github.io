## 一、原理

从流对象加载 AB 包，比如文件流、内存字节流、网络流、解密流等等。它的一个好处是自定义数据来源，比较灵活。第二个好处是，相比 LoadFromMemory 内存开销更小，使用 LoadFromMemory 需要提前从磁盘读取文件的字节序列，放在内存中；而 LoadFromStream 由于从流中加载（只为流对象分配一个小缓冲区），相当于直接读磁盘数据来创建 AB 对象，节省了一些内存开销。

## 二、优点

- 灵活性高，可自定义数据源
- 避免双倍内存（比LoadFromMemory好！）
- 支持边读边处理（解密、解压等），适合加密场景

## 三、缺点

- 相比LoadFromFile稍慢
- 需要手动管理Stream生命周期
- 代码复杂度较高

## 四、代码示例

### 1. LoadFromStream

```cs
void LoadABFromStream()
{
    string path = Path.Combine(
        Application.streamingAssetsPath,
        "myassetbundle"
    );

    // 创建文件流，无需将整个 AB 文件加载进内存
    using (FileStream stream = new FileStream(
        path,
        FileMode.Open,
        FileAccess.Read))
    {
        // 从流加载AB
        AssetBundle ab = AssetBundle.LoadFromStream(stream);

        if (ab != null)
        {
            GameObject prefab = ab.LoadAsset<GameObject>("MyPrefab");
            Instantiate(prefab);

            Debug.Log("从流加载AB成功");

            // 使用完后卸载
            ab.Unload(false);
        }
    } // using 会自动关闭stream
}
```

### 2. LoadFromStreamAsync

```cs
IEnumerator LoadABFromStreamAsync()
{
    string path = Path.Combine(
        Application.streamingAssetsPath,
        "myassetbundle"
    );

    using (FileStream stream = new FileStream(
        path,
        FileMode.Open,
        FileAccess.Read))
    {
        // 异步从流加载
        AssetBundleCreateRequest request =
            AssetBundle.LoadFromStreamAsync(stream);

        yield return request;

        AssetBundle ab = request.assetBundle;

        if (ab != null)
        {
            GameObject prefab = ab.LoadAsset<GameObject>("MyPrefab");
            Instantiate(prefab);
        }
    }
}
```

### 4. 使用自定义解密流

这是 LoadFromStream 的**最佳应用场景**：边读边解密

```cs title="创建一个自定义的XOR解密流"
using System.IO;

/// <summary>
/// XOR解密流 - 边读边解密
/// </summary>
public class XORDecryptStream : Stream
{
    private Stream baseStream;
    private byte key;

    public XORDecryptStream(Stream baseStream, byte key)
    {
        this.baseStream = baseStream;
        this.key = key;
    }

    /// <summary>
    /// 读取数据并自动解密
    /// </summary>
    public override int Read(byte[] buffer, int offset, int count)
    {
        // 从底层流读取数据
        int bytesRead = baseStream.Read(buffer, offset, count);

        // 边读边解密（XOR运算）
        for (int i = 0; i < bytesRead; i++)
        {
            buffer[offset + i] ^= key;
        }

        return bytesRead;
    }

    // 实现Stream的必需属性和方法
    public override bool CanRead => true;
    public override bool CanSeek => baseStream.CanSeek;
    public override bool CanWrite => false;
    public override long Length => baseStream.Length;

    public override long Position
    {
        get => baseStream.Position;
        set => baseStream.Position = value;
    }

    public override void Flush() => baseStream.Flush();

    public override long Seek(long offset, SeekOrigin origin)
    {
        return baseStream.Seek(offset, origin);
    }

    public override void SetLength(long value)
    {
        throw new System.NotSupportedException();
    }

    public override void Write(byte[] buffer, int offset, int count)
    {
        throw new System.NotSupportedException();
    }

    protected override void Dispose(bool disposing)
    {
        if (disposing)
        {
            baseStream?.Dispose();
        }
        base.Dispose(disposing);
    }
}
```

```cs title="使用这个自定义解密流"
using UnityEngine;
using System.Collections;
using System.IO;

public class CustomStreamLoader : MonoBehaviour
{
    void Start()
    {
        StartCoroutine(LoadEncryptedAB());
    }

    IEnumerator LoadEncryptedAB()
    {
        string encryptedPath = Path.Combine(
            Application.streamingAssetsPath,
            "encrypted.ab"
        );

        // 创建文件流 → 解密流 → 加载AB
        using (FileStream fileStream = new FileStream(
            encryptedPath,
            FileMode.Open,
            FileAccess.Read))
        using (XORDecryptStream decryptStream = new XORDecryptStream(
            fileStream,
            123))  // 解密密钥
        {
            // 边读边解密，无双倍内存！
            AssetBundleCreateRequest request =
                AssetBundle.LoadFromStreamAsync(decryptStream);

            yield return request;

            AssetBundle ab = request.assetBundle;

            if (ab != null)
            {
                GameObject prefab = ab.LoadAsset<GameObject>("MyPrefab");
                Instantiate(prefab);

                Debug.Log("从加密AB加载成功!");
            }
        }
    }
}
```