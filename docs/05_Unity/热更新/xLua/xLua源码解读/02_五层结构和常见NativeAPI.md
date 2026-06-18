## 五层结构

整个 xLua 互操作链上，可以将结构分为五层：

- Lua 业务代码
- xLua native
- ObjectTranslator
- Wrap / Bridge
- C# 业务代码

### 1. Lua 业务代码

这层负责 Lua 一侧的业务逻辑，通过 CS.{NameSpace}.{Class}.{Method} 这种形式访问 CS 全局表来访问 C# ，或者已经持有一个 C# 对象，通过 {CS_OBJ}.{Method/Field} 来访问 C# 。

### 2. xLua native

这层指的是 `xlua.dll` 或者 `libxlua.so` 这种 xLua 提供的库。它包含 Lua 内核以及 xLua 提供的扩展逻辑。这层的主要作用是提供 `lua_push***` / `lua_to***` / `xlua_pushcsobj` 等 native API 。

### 3. ObjectTranslator

对应 xLua 提供的 `ObjectTranslator.cs` 脚本，它是核心翻译层。维护「C# 对象池 <-> Lua userdata」的映射，提供 Push / Get / PushAny / PushByType / FastGetCSObj 等通用 API。

### 4. Wrap / Bridge

它对应这些文件：

``` title="Wrap / Bridge"
Assets/xLuaWraps/...（生成的 Wrap 代码）
xlua/Src/DelegateBridge.cs
xlua/Src/StaticLuaCallbacks.cs
```

每个被 [LuaCallCSharp] 标记的 C# 类型在编译期生成一份 Wrap，这些 Wrap 文件把「Lua 栈参数」翻译成「C# 方法调用」，再把返回值压回 Lua 栈。它们是在 Lua 调用 C# 是发挥巨大作用。

DelegateBridge 是 Lua Function 的 C# Delegate 代理对象，它持有 Lua 函数引用，在 C# 委托被 Invoke 时负责恢复 Lua 函数并完成调用。

两者的方向有点“镜像”的意味，Wrap 负责对「C# 方法调用」进行一层封装，便于 Lua 调用；而 Bridge 则相反，它对「Lua 函数调用」进行封装，便于 C# 调用。

### 5. C# 业务代码

负责 C# 一侧的业务逻辑，可以通过 `luaEnv.DoString("{LuaCode}")` 来直接执行一段 Lua 代码，也可以通过 `LuaTable.Get`、`LuaTable.GetDelegate`、`LuaTable.GetLuaFunction` 等方法来间接访问 Lua 。

## 常见 Native API

为了方便看懂 xLua 如何操作 Lua VM Stack，这里例举一些常见 Native API 的作用，只说明大概逻辑，并不深度分析源码，能保证我们看懂 xLua 中间的运作流程即可。如果需要深究 xlua native，可以自己想办法逆向 `xlua.dll` 或者 `libxlua.so` 。也可以看 `LuaJIT` ，但 `LuaJIT` 缺少 xLua 扩展的那部分逻辑。


