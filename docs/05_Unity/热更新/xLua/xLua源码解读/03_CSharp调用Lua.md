C# 调用 Lua 有多种方式，但归根结底可以分为两种：

- LuaEnv.DoString 直接执行一段 Lua 代码
- 通过 LuaTable.Get 获得一个整数句柄，通过这个句柄可以进一步访问 Lua 的对象或者方法

## LuaEnv.DoString

`DoString` 有两个版本，传入字符串的版本是对字节数据版本的便利性封装

```cs title="DoString的函数定义"
// 字符串版本
public object[] DoString(
    string chunk, string chunkName = "chunk", LuaTable env = null)
{
    byte[] bytes = System.Text.Encoding.UTF8.GetBytes(chunk);
    return DoString(bytes, chunkName, env);
}

// 字节数组版本
public object[] DoString(
    byte[] chunk, string chunkName = "chunk", LuaTable env = null)
{
#if THREAD_SAFE || HOTFIX_ENABLE
    lock (luaEnvLock)
    {
#endif
        var _L = L;
        int oldTop = LuaAPI.lua_gettop(_L);
        int errFunc = LuaAPI.load_error_func(_L, errorFuncRef);
        if (LuaAPI.xluaL_loadbuffer(_L, chunk, chunk.Length, chunkName) == 0)
        {
            if (env != null)
            {
                env.push(_L);
                LuaAPI.lua_setfenv(_L, -2);
            }

            if (LuaAPI.lua_pcall(_L, 0, -1, errFunc) == 0)
            {
                LuaAPI.lua_remove(_L, errFunc);
                return translator.popValues(_L, oldTop);
            }
            else
                ThrowExceptionFromError(oldTop);
        }
        else
            ThrowExceptionFromError(oldTop);

        return null;
#if THREAD_SAFE || HOTFIX_ENABLE
    }
#endif
}
```

步骤剖析：

=== "Step 1"

    ![Step 1](xLua_01.png)

=== "Step 2"

    ![Step 2](xLua_02.png)

=== "Step 3"

    ![Step 3](xLua_03.png)

=== "Step 4"

    ![Step 4](xLua_04.png)

=== "Step 5"

    ![Step 5](xLua_05.png)

=== "Step 6"

    ![Step 6](xLua_06.png)

=== "Step 7"

    ![Step 7](xLua_07.png)

关于 `translator.popValues` 这个函数，目前只用知道它大概做了哪些事即可。后续会逐渐了解 `ObjectTranslator` 的设计。

## LuaTable.Get

通过这个调用，我们可以在 C# 一侧得到 Lua 对象或者方法的整数句柄，先了解清楚这个机制，后续才能明白 C# 调用 Lua 是怎么运作的。

`LuaTable.Get` 有一些多态版本，但最核心的是下面这个版本的实现，其他还有装箱版本，但现在也废弃了，不提倡使用。

```cs title=""
public void Get<TKey, TValue>(TKey key, out TValue value)
{
#if THREAD_SAFE || HOTFIX_ENABLE
    lock (luaEnv.luaEnvLock)
    {
#endif
        var L = luaEnv.L;
        var translator = luaEnv.translator;
        int oldTop = LuaAPI.lua_gettop(L);
        LuaAPI.lua_getref(L, luaReference);
        translator.PushByType(L, key);

        if (0 != LuaAPI.xlua_pgettable(L, -2))
        {
            string err = LuaAPI.lua_tostring(L, -1);
            LuaAPI.lua_settop(L, oldTop);
            throw new Exception("get field [" + key + "] error:" + err);
        }

        LuaTypes lua_type = LuaAPI.lua_type(L, -1);
        Type type_of_value = typeof(TValue);
        if (lua_type == LuaTypes.LUA_TNIL && type_of_value.IsValueType())
        {
            throw new InvalidCastException("can not assign nil to " + type_of_value.GetFriendlyName());
        }

        try
        {
            translator.Get(L, -1, out value);
        }
        catch (Exception e)
        {
            throw e;
        }
        finally
        {
            LuaAPI.lua_settop(L, oldTop);
        }
#if THREAD_SAFE || HOTFIX_ENABLE
    }
#endif
}
```

在开始之前，先讲一下 `luaReference` 是什么。它是 C# 侧的 LuaTable 对象对 Lua 侧某个对象的引用，本质上是一个整数句柄。由于两侧 GC 不同，直接持有 Lua 对象是不合理的，所以 xLua 把这个 table 存到 Lua 注册表（LUA_REGISTRYINDEX）里，返回这个索引作为句柄。之后在 C# 这边只持有这个句柄，要访问这个 Lua 对象时通过调用 `lua_getref` 把 Lua 对象压入栈。

步骤剖析：

=== "Step 1"

    ![Step 1](xLua_08.png)

=== "Step 2"

    ![Step 2](xLua_09.png)

=== "Step 3"

    ![Step 3](xLua_10.png)

=== "Step 4"

    ![Step 4](xLua_11.png)

=== "Step 5"

    ![Step 5](xLua_12.png)

Lua 的所有对象本质上都是一个 Table，而执行一个特定的 Lua 逻辑，本质上就是 `LuaTable.LuaFunction()`，有了 `LuaTable.Get` 既可以获取一个 LuaTable，又可以从 LuaTable 中获取指定的 LuaFunction ，所以 C# 调用 Lua 是可行的。现在比较常用的方式有两种：

- **Get<LuaFunction\>**：将获得的 Lua 函数作为 LuaFunction 使用
- **Get<强类型委托>**：比较推荐的做法，自定义委托类型

下面逐个讲解。

## 获取 LuaFunction 并调用

整个流程分两个阶段：获取和调用

### 1. 获取 LuaFunction

这部分就是 LuaTable.Get ，泛型传的是 `LuaFunction`，不过这里要细讲下 `translator.Get(L, -1, out value);` 是如何返回一个 `LuaFunction` 对象的。

=== "Step 1"

    ```cs title="ObjectTranslator.cs:957"
    // 泛型分发：快路径 vs 慢路径
    public void Get<T>(RealStatePtr L, int index, out T v)
    {
        Func<RealStatePtr, int, T> get_func;
        if (tryGetGetFuncByType(typeof(T), out get_func))
        {
            v = get_func(L, index); // 快路径：直接调对应的 lua_toxxx
        }
        else
        {
            v = (T)GetObject(L, index, typeof(T)); // 慢路径：走通用 caster
        }
    }
    ```

    !!! note "注解"
        基础类型（包括byte[]、IntPtr）走的快路径，直接 P/Invoke 得到返回值；而向 LuaTable、LuaFunction、自定义类等走慢路径，这里我们是 `LuaFunction`，所以继续看慢路径。

=== "Step 2"

    ```cs title="ObjectTranslator.cs:925"
    // 看是不是 userdata
    public object GetObject(RealStatePtr L, int index, Type type)
    {
        int udata = LuaAPI.xlua_tocsobj_safe(L, index);

        if (udata != -1) // 是有效的 C# 引用类型对象
        {
            // 从对象池拿出来，返回
            object obj = objects.Get(udata);
            RawObject rawObject = obj as RawObject;
            return rawObject == null ? obj : rawObject.Target;
        }
        else
        {
            // 除了 C# 引用类型对象，userdata 还可能是其他 C# 值类型，检查一下
            if (LuaAPI.lua_type(L, index) == LuaTypes.LUA_TUSERDATA)
            {
                GetCSObject get;
                int type_id = LuaAPI.xlua_gettypeid(L, index);
                if (type_id != -1 && type_id == decimal_type_id)
                {
                    // 是 decimal 类型
                    decimal d;
                    Get(L, index, out d);
                    return d;
                }
                Type type_of_struct;
                if (type_id != -1 && typeMap.TryGetValue(type_id, out type_of_struct) && type.IsAssignableFrom(type_of_struct) && custom_get_funcs.TryGetValue(type, out get))
                {
                    // 是别的 C# 值类型，比如
                    // 1. C# struct
                    // 2. 自定义值类型
                    // 3. 其它通过 custom_get_funcs 注册的类型
                    return get(L, index);
                }
            }
            // 都没命中，走通用转换器
            return (objectCasters.GetCaster(type)(L, index, null));
        }
    }
    ```

    !!! note "注解"
        前面讲过 C# 持有 Lua 对象实际上是持有 Lua 注册表的整数句柄，并没有真的持有 Lua 对象。反过来，这里有检查 Lua 持有 C# 对象的情况，这里也类似，Lua 也仅仅持有一个 int 索引，只不过这个索引被包了一层变成了一个 userdata ，userdata 体里只存了那个 int 索引。这里的 `objects` 类似 Lua 的注册表，通过这个 int 索引就可以取对应的 C# 对象。
        这里我们取的是一个 Lua 原生的 function，并不是 userdata，所以走的是通用转换器。

## 使用强类型委托（推荐）