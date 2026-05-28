## 先下定义

GF 的文件系统模块比较特殊，`FileSystemManager` 按照路径名称管理着多个 `FileSystem`, 单个 `FileSystem` 和传统意义上的文件系统不一样，它更像是一个“散文件的打包容器”，然而它和传统打包文件又不一样，它没有 zip 操作，只是单纯地将这些散文件的二进制流按照一定规则编排在一起，实现高效的读、写、重命名、删除等操作，在完成这些的同时还做到了尽量复用空闲空间。

``` text title="FileSystem做的事"
一个物理文件（例如 xxx.dat）
    -> 里面放很多逻辑文件
    -> 用文件名查到块信息
    -> 再按偏移从容器里读出真实字节
```

## 职责分层

``` text title="各个角色的职责"
IFileSystemManager
    -> FileSystemManager
        -> 管理多个 FileSystem 实例
        -> 负责 Create / Load / Destroy 单个 FileSystem 实例

IFileSystem
    -> FileSystem
        -> 真正的容器实现
        -> 管理磁盘布局、元数据、读写、回收
        -> 内部 3 个关键结构：
            -> HeaderData   头信息
            -> BlockData    块信息
            -> StringData   文件名字串信息

IFileSystemHelper
    -> 负责创建 FileSystemStream

FileSystemStream
    -> 抽象流接口
    -> CommonFileSystemStream 是默认基于 FileStream 的实现
```

可以把职责拆成两层来看：

- `FileSystemManager` 解决“生命周期管理”和“多个文件系统实例管理”
- `FileSystem` 解决“一个容器文件内部到底怎么存、怎么读、怎么写”

## 接口设计

来看看接口对外承诺了些什么

### 1. IFileSystemManager

这是模块入口，主要提供：

1. `SetFileSystemHelper`
2. `CreateFileSystem`
3. `LoadFileSystem`
4. `DestroyFileSystem`
5. `GetFileSystem`
6. `GetAllFileSystems`

它的作用很像一个“文件系统实例仓库”，就是负责管理维护多个 `FileSystem` 实例。

### 2. IFileSystem

这是单个 `FileSystem` 暴露出来的能力，主要有：

1. 查询：`HasFile`、`GetFileInfo`
2. 读取：`ReadFile`、`ReadFileSegment`
3. 写入：`WriteFile`
4. 导出：`SaveAsFile`
5. 改名：`RenameFile`
6. 删除：`DeleteFile`

所以对外部调用者来说，`FileSystem` 看起来就像一个“简化版虚拟文件系统”。

### 3. FileSystemAccess

访问模式只有 3 种：

1. `Read`
2. `Write`
3. `ReadWrite`

这决定了后续哪些操作允许执行。

## FileSystem 的磁盘布局

``` text title="磁盘布局"
+---------------------------+  ---+
| HeaderData                |     |
+---------------------------+     |
| BlockData[MaxBlockCount]  |     |
| (BlockData[0])            |     |
| (BlockData[1])            |     |
| (BlockData[2])            |     |
| (...         )            |     元
+---------------------------+     数
| StringData[MaxFileCount]  |     据
| (StringData[0])           |     区
| (StringData[1])           |     |
| (StringData[2])           |     |
| (...          )           |     |
+---------------------------+     |
| 对齐填充到 4 KB 边界      |     |
+---------------------------+  ---+
| 文件内容区                |
| Cluster A data           |
| Cluster B data           |
| Cluster C data           |
| ...                       |
+---------------------------+
```

首先注意这并不是内存布局，这是单个 FileSystem 落地到磁盘的文件内容布局，在代码中体现在对 `FileStream` 的操作中。

### 1. 元数据区

创建 FileSystem 时，元数据区不是按需一点点扩充起来的，而是一次性预留：

- 头信息区，单个 HeaderData
- 块描述区，`MaxBlockCount` 个 BlockData 槽位
- 文件名区，`MaxFileCount` 个 StringData 槽位
- 可能存在的填充区，用于对齐到 4KB 的边界

其中块描述区和文件名区的长度可以由外部指定，它们会直接影响这个容器文件刚创建出来时的物理大小

### 2. HeaderData

HeaderData 记录了整个容器的当前情况，它主要有：

- 文件头标记 `GFF`，猜测是 GameFramework FileSystem 的简写
- 版本号，当前是 `0`
- 4 字节随机加密码，用于文件名异或
- 文件名区的槽位数量 `MaxFileCount`
- 块描述区的槽位数量 `MaxBlockCount`
- 当前已经实际使用了多少个 `BlockData` 槽位 `BlockCount`

### 3. BlockData

单个逻辑文件最后会对应一个 `BlockData`，里面有：

- `StringIndex`：文件名在 `StringData` 区里的索引
- `ClusterIndex`：数据从哪个 cluster 开始
- `Length`：文件真实长度

`BlockData` 有三种状态：

- 使用中 Using
- 空闲中 Free
- 空白槽 Empty

它们的数据状态大概是这样的：

=== "使用中"
    ``` title="Using"
    // 这代表它指向一个有效的文件
    StringIndex = 2, // 文件名存储在文件名区 index=2 的槽位
    ClusterIndex = 4, // 文件内容从文件内容区 index=4 的 Cluster 开始
    Length = 2000, // 文件实际长度是 2000 个字节
    ```
=== "空闲中"
    ``` title="Free"
    // 这代表它没有指向一个有效的文件，但指向一个空闲的 Cluster
    // 这个 Cluster 没有存储任何有效数据
    StringIndex = -1, // 小于0代表是该槽位指向的文件内容区没有被使用
    ClusterIndex = 4, // 空闲区域从文件内容区 index=4 的 Cluster 开始
    Length = 8192, // 空闲区域长度是 8192 个字节，占两个 Cluster
    ```
=== "空白槽"
    ``` title="Empty"
    // 这代表这个槽位自己本身没有被使用，甚至没有指向空闲块
    StringIndex = -1, // 没有被使用
    ClusterIndex = 0, // 没有被使用
    Length = 0, // 没有被使用
    ```

为什么会同时有“空闲中”和“空白槽”两种状态？这是因为 FileSystem 可以将空闲的文件内容区合并起来。比如文件内容区有两个空闲的 4KB 的 Cluster，恰好他们相邻可以合并成一个大的 8KB 的区域，那么合并后，只需要其中一个 BlockData 指向这个大区域即可，那么另外一个 BlockData 自然就不需要指向任何 Cluster，就变成“空白槽”了

### 4. StringData

`StringData` 专门存文件名。

它的结构是：

- `m_Length`：名字长度
- `m_Bytes[255]`：最多 255 字节的 UTF-8 数据

这里注意两个事实：

1. 文件名上限本质上是 255 个 UTF-8 字节，不是 255 个字符。
2. 它对文件名做了一个简单 XOR 处理，只是轻量混淆，不是安全加密。

也就是说：

- 容器里的“文件内容”并没有被 `FileSystem` 自动加密。
- 被 XOR 处理的是“文件名元数据”。

文件内容如果要加密解密，是上层模块的职责。

### 5. Cluster

文件内容区由一个个的 Cluster 排列而成，源码里有一个常量：

```csharp
private const int ClusterSize = 1024 * 4;
```

这说明文件内容实际分配时按照 4KB 对齐，所以就算一个大小为 1 字节的文件，底层也会占用 4KB，一个 5000 字节的文件则向上对齐，占用 8KB 。

## FileSystem 的成员

### 5.1 `m_FileDatas`

类型是 `Dictionary<string, int>`，负责记录“文件名-BlockIndex”的映射，这是最核心的查找表。

读文件时第一步就是：文件名 -> blockIndex -> BlockData -> 偏移 + 长度

### 5.2 `m_BlockDatas`

类型是 `List<BlockData>`，它保存所有块描述符。每个逻辑文件、每个空闲块，都体现在这里。

### 5.3 `m_FreeBlockIndexes`

类型是 `GameFrameworkMultiDictionary<int, int>`，长度 -> 多个空闲 blockIndex，这是空间管理的关键。

它不是按“地址”找空闲块，而是按“长度”找空闲块。这样写入新文件时，就能找一个“足够大且尽量小”的空闲块复用。比如现在字典内容如下：

``` title="m_FreeBlockIndexes假设有如下内容"
8192 -> [4]、[3]
4096 -> [5]、[10]
0    -> [2]
```

当前我需要一个 3000 字节的空间，那么需要一个大于 3000 的空间，4096 和 8192 都满足，但 4096 更小，所以我会找到 [5] 号位置的空闲块

### 5.4 `m_StringDatas`

类型是 `SortedDictionary<int, StringData>`，stringIndex -> StringData，它保存当前正在使用的文件名记录。

### 5.5 `m_FreeStringIndexes` 和 `m_FreeStringDatas`

这两个队列负责复用已删除文件留下的文件名槽位和字节数组，减少重复分配。