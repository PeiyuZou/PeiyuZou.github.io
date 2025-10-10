## Native Memory 介绍

### 1. Allocator与Memory Label

Unity 在里面重载了 C++ 的所有分配内存的操作符，例如 alloc，new 等。每个操作符在被使用的时候要求有一个额外的参数就是 Memory Label，在 Memory Profiler （Windows/Analysis/Memory Profiler）中查看 Memory Details 里的 Name 很多就是 Memory Label。它指的就是当前的这一块内存内存要分配到哪个类型池里。

### 2. GetRuntimeMemory

Unity 在底层会用 Allocator，使用重载过的分配符分配内存的时候，会根据 Memory Label 分配到不同的 Allocator池 里面。每个 Allocator 池，单独做自己的跟踪。当要在 Runtime 去 Get 一个 Memory Label 下面池的时候，可以从对应的 Allocator 中取，可以从中知道有什么东西，有多少兆。

### 3. NewAsRoot

前面提到的 Allocator 的生成是使用 NewAsRoot，生成一个所谓的 Memory Island，它下面会有很多的子内存。例如一个 Shader，当加载一个 shader 进内存的时候，首先会生成一个 shader 的 Root，也就是 Memory Island。然后 Shader 底下的数据，例如 Subshader，Pass，Parameters 等，会作为该 Root 底下的成员，依次的分配。所以最后统计 Runtime 的内存时，统计这些 Root 即可。

### 4. 返还操作系统

因为是 C++ 的，所以当 delete 或 free 一个内存的时候，会立刻返回给系统。这和托管内存不一样，托管内存需要 GC 后才返回。

## Native Memory 最佳实践

在使用Unity的时候，如果某些方面使用不当，会造成 Native Memory 的增长，所以这部分也是可以优化调整的。

### 1. Scene

首当其冲，最常见的导致 Native Memory 增长的原因，就是Scene。因为 Unity 是 C++ 引擎，所有的实体最终都会反映在 C++ 上，而不会反映在托管堆上。所以当 Scene 构建一个 GameObject 的时候，实际上在 Unity 的底层会构建一个或多个 Object 来存储这一个 GameObject 的信息（Component信息等）。所以当一个 Scene 里面有过多的 GameObject 存在的时候，Native Memory 就会显著的上升，甚至可能导致内存溢出。

!!! note "注意"
    所以这里有一个经验之谈：当发现 Native Memory 大量上升时，首先去着重检查你的 Scene

### 2. Audio

#### 2.1 DSP buffer

指一个声音的缓冲，当一个声音要播放的时候，需要向 CPU 去发送指令。如果声音的数据量非常的小，会造成频繁的向 CPU 发指令，造成 IO 压力。在 Unity 的 FMOD 声音引擎里面，一般会有一个 Buffer，当 Buffer 填充满了才会去向 CPU 发送一次播放声音的指令。所以 DSP Buffer大小的设置非常考究，一般会导致两种问题：

- 设置的值过大，会导致声音的延迟，因为填充满需要很多的声音数据，当我们声音数据不大的时候，就会产生延时。
- 设置的值过小，会导致 CPU 负担上升，因为会频繁的发送。

#### 2.2 Audio Import Settings

- **Force To Mono**：​这个选项作用是强制单声道，很多声音为了追求质量会设置成双声道，导致声音在包体和内存中，占用的空间加倍，但是95%以上的声音，两个声道是完全一样的数据。因此对声音不是很敏感的项目建议勾选此项，来降低内存的占用。
- **Compression Format**：不同的平台有不同的声音格式的支持，iOS 对 MP3 有硬件支持，Android 暂时没有硬件支持。建议 iOS 使用 ADPCM 和 MP3 格式，Android 使用 Vorbis 格式。
- **Load Type**：决定声音在内存中的存在形态：
    - Decompress On Load：当audio clip被加载时，解压声音数据，适用于小型音频文件（< 200kb）
    - Compressed In Memory：声音数据将以压缩的形式保存在内存当中，适用于中型音频文件（>= 200kb）
    - Streaming：从磁盘读取声音数据，适用于大型音频文件，例如背景音

!!! note "注意"
    - Decompress On Load，要求文件必须小于 200kb，因为内部内存管理的问题，如果是大于 200kb 的文件，那么也还是只会被分配到不足 200kb 的内存。
    - Bitrate：可以对音频文件本身进行压缩，降低文件的比特率（bitrate），前提音频品质不会被破坏太严重。


### 3. Code Size

代码文件也是占内存的，需要加载进内存执行。一个典型的例子是模板泛型的滥用，例如一个模板函数有四五个不同的泛型参数，类型也不尽相同（float，int，double等），最后展开得到的一个 cpp 文件可能会很大。因为实际上 C++ 编译的时候用的所有的 Class，所有的 Template 最终都会被展开成静态类型。因此当模板函数有很多排列组合时，最后编译会得到所有的排列组合代码，导致文件很大。

这不光会影响到最终代码文件的大小，导致 Native Memory 间接增长，同时也会影响IL2CPP编译速度，接触过 C++ 编译应该知道，单一一个 cpp 文件编译的话是没办法并行的，只能单核处理，所以也间接地导致打包慢。

### 4. AssetBundle

#### 4.1 TypeTree

这个东西存在是为了做这件事：Unity前后有很多的版本，不同的版本中很多的类型可能会有数据结构的改变，为了做数据结构的兼容，会在生成数据类型序列化的时候，顺便生成一个叫 TypeTree 的东西。就是当前这个版本用到了哪些变量，它们对应的数据类型是什么，当进行反序列化的时候，根据 TypeTree 去做反序列化。如果上一个版本的类型在这个版本没有，那 TypeTree 里就没有它，所以不会去碰到它。如果有新的 TypeTree，但是在当前版本不存在的话，那要用它的默认值来序列化。从而保证了在不同版本之间不会序列化出错。

在构建 AssetBundle 的时候，可以通过以下代码关掉 TypeTree 的生成：

```  { .cs .copy .annotate }
BuildAssetBundleOptions.DisableWriteTypeTree
```

什么时候可以关呢？当你可以保证构建 AssetBundle 的 Unity版本和使用它的 Unity 的版本是一模一样的时候（对兼容性不会有影响），就可以关闭。这样有三个好处：一、可以减少内存；二、AssetBundle 包大小会减少；三、build 和 Runtime 会变快，因为不会去序列化和反序列化 TypeTree（如果开了 TypeTree，序列化会做两步，首先去序列化 TypeTree，然后再去序列化实际的东西，反序列化也一样）

#### 4.2 压缩方式（Lz4 和 Lzma）

Unity 目前主推 Lz4（也就是ChunkBased，BuildAssetBundleOptions.ChunkBasedCompression），Lz4 非常快，大概是 Lzma 的十倍以上的速度，但平均压缩比例比 Lzma 差 30% 左右，即包体更大。但 Lzma 基本可以不用了，因为 Lzma 解压和读取速度都非常慢，并且内存占比高，因为它的读取不是基于 ChunkBased，而是 Stream，也就是一次全解压出来。ChunkBased 可以逐块解压，每次解压可以重用之前的内存，减少内存的峰值。

!!! note "Lz4开源仓库"
    Lz4目前是开源的，可以了解下它的原理：https://github.com/lz4/lz4

#### 4.3 大小和数量

AssetBundle 分两部分，一部分是头（用于索引），一部分是实际的打包的数据部分。如果每个 Asset 都单独打成一个 AssetBundle，那么可能所有问题加起来头的部分比数据还大。所以这个大小不适合太大也不能太小，官方建议一个AssetBundle在 1-2M，但是现在进入 5g 时代的话，可以适当加大，因为网络带宽更大了。

### 5. Resources

如果使用 Resources 模式打包，Resources 文件夹里的内容被打进包的时候会做一个红黑树（R-B Tree）用做索引，即检索资源到底在什么位置。所以Resource越大，红黑树越大，它不可卸载，并在刚刚加载游戏的时候就会被一直加在内存里，极大的拖慢游戏的启动时间，因为红黑树没有分析和加载完，游戏是不会启动的，并造成持续的内存压力。所以建议不要使用Resource，使用AssetBundle。

### 6. Texture

- Upload Buffer：和声音的Buffer类似，填满后向 GPU push 一次
- Read/Write：没必要的话就关闭，正常情况，Texture 读进内存解析完了搁到 Upload Buffer 里之后，内存里那部分就会 delete 掉。除非开了Read/Write，那就不会 delete 了，会在显存和内存里各一份。前面说过手机内存显存通用的，所以内存里会有两份。
- ​Mip Maps：例如 UI 元素这类相对于相机Z轴的值不会有任何变化的纹理，关闭该选项。
- Alpha Source：对于不透明纹理，关闭其alpha通道。
​​- Max Size：根据平台不同，纹理的Max Size设成该平台最小值。
- POT：纹理的大小尽量为2的幂次方（POT），因为有些压缩格式可能不支持非2的幂次方的。
- 压缩格式：
    - Android
        - 支持 OpenGL ES 3.0 的使用 ETC2，RGB 压缩为 RGB Compressed ETC2 4bits，RGBA 压缩为 RGBA Compressed ETC2 8bits
        - 需要兼容 OpenGL ES 2.0 的使用 ETC，RGB 压缩为 RGB Compressed ETC 4bits，RGBA 压缩为 RGBA 16bits。（压缩大小不能接受的情况下，压缩为 2 张 RGB Compressed ETC 4bits）
        - 目前主流正在从 ETC2 向 ASTC 转变，后者的压缩质量和大小上都有优势。它支持 OpenGL ES 3.1 和部分 OpenGL ES 3.0 的设备。目前市面上绝大多数安卓设备都支持，已经可以普及使用
    - iOS
        - 支持 OpenGL ES 3.0 的使用ASTC，RGB 压缩为RGB CompressedASTC 6x6 block，RGBA 压缩为 RGBA Compressed ASTC 4x4 block。
        - 对于法线贴图的压缩精度较高可以选择 RGB CompressedASTC 5x5 block。
        - 需要兼容OpenGLES 2.0 的使用 PVRTC，RGB 压缩为PVRTC 4bits，RGBA压缩为 RGBA 16bits。（压缩大小不能接受的情况下，压缩为 2 张 RGB Compressed PVRTC 4bits）
        - 截止目前 ASTC 合并了 RGB 和 RGBA 格式，压缩格式的选择上面更加方便了

### 7. Mesh

- Read/Write：同Texture，若开启，Unity会存储两份Mesh，导致运行时的内存用量变成两倍。
- Compression：Mesh Compression 是使用压缩算法将Mesh数据进行压缩，结果是会减少占用硬盘的空间，但是在 Runtime 的时候会被解压为原始精度的数据，因此内存占用并不会减少。需要注意的是有些版本开了，实际解压之后内存占用大小会更严重。
- Rig：如果没有使用动画，请关闭Rig，例如房子，石头这些。
- ​​Blendshapes：如果没有用到Blendshapes，也关闭。
- Material设置：如果Material没有用到法向量和切线信息，关闭可以减少额外信息。

### 8. Assets

和整个的 Asset 管理有关系，Unity 官网上有关于资源管理的文章，找到再补充。