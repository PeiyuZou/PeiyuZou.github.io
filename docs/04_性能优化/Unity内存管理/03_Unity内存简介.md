## Unity引擎概述

Unity 是一个 C++ 引擎，并不是 C# 引擎，底层代码全部是由 C++ 写的，除了一些 Editor 里面的 Services 可能会用到 NodeJS 这些网络的语言，Runtime 里面用到的每行 Unity 底层代码都是 C++ 的。

Unity 实际上分为三层：

- 最底层是 Runtime，全是 Native C++ 代码。
- 最上层是 C#，Unity 本身也有一些 C#，例如 Unity 的 Editor 是用 C# 写的，还有些 Package 也是 C# 写的。
- 中间还有一层叫 Binding，可以看见很多的 .bindings.cs 文件（基于 C# 的 binding 语言，一开始是 Unity 自定义的一种语言），这些文件的作用就是把 C++ 和 C# 联系在一起，为 C# 层提供所有的 API。因此使用 Unity 时看见的 C# API，都是在 Binding 层中自定义的。这些文件底层运行的时候还是 C++，只是个 Wrapper（封装）。

最早用户代码是运行在 C# 上，是 MonoRuntime。现在可以通过 IL2CPP 将其转成 C++ 代码，所以现在几乎没有纯正的 C# 在运行了。

Unity 的 VM（虚拟机：Virtual Machine）依旧还是存在，主要用于跨平台，有了一层 VM 抽象后，跨平台的工作会容易很多，IL2CPP 本质也是 VM 。

## Unity 的内存

Unity的内存，可以从以下几个层面去理解

### 1. 按照分配方式划分

按照分配方式分为：Native Memory（原生内存）和 Managed Memory（托管内存）。Native Memory 并不会被系统自动管理，需要手动去释放。而 Managed Memory 的内存管理是自动的，会通过 GC 来释放。

此外 Unity 在 Editor 和 Runtime 下，内存的管理方式是不同的，除了内存大小不同，内存的分配时机以及分配方式也不同。例如 Asset，在Runtime 时，只有用户代码 Load 的时候才会进内存。而 Editor 模式下，为了编辑的便利性，只要打开 Unity 就会进内存（所以打开很慢）。后续有推出 Asset Pipeline 2.0，一开始导入一些基本的 Asset，剩下的 Asset 只有使用的时候才会导入，这样即使是很大的工程，也可以尽量减少使用者对不关心的 Asset 付出导入时间的代价。

### 2. 按照内存管理方式划分

按照管理方式分为：引擎管理内存和用户管理内存。引擎管理内存即引擎运行的时候分配的一些内存，例如很多的 Manager 和 Singleton ，这些内存开发者一般是碰触不到的。用户管理内存也就是开发者开发时使用到的内存，是我们平时接触最多的部分。

### 3. Unity 监测不到的内存

- 用户分配的 Native 内存。比如自己写的 Native 插件（C++插件）导入 Unity ，这部分 Unity 是检测不到的，因为 Unity 没法分析已编译的 C++ 是如何分配和使用内存的
- Lua ，它完全自己管理的，Unity 也没法统计到它内部的情况