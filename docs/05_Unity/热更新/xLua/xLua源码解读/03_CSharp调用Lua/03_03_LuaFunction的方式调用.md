整个流程分两个阶段：获取和调用

## 1. 获取 LuaFunction

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

=== "Step 3"

    ```cs title="ObjectCasters.cs:725"
    public ObjectCast GetCaster(Type type)
    {
        if (type.IsByRef) type = type.GetElementType();

        Type underlyingType = Nullable.GetUnderlyingType(type);
        if (underlyingType != null)
            return genNullableCaster(GetCaster(underlyingType));

        ObjectCast oc;
        if (!castersMap.TryGetValue(type, out oc))
        {
            oc = genCaster(type); // 没有就动态生成一个
            castersMap.Add(type, oc);
        }
        return oc;
    }
    ```

    !!! note "注解"
        这里我们传入的类型是 `LuaFunction`，在 `castersMap` 中已经预先注册好获取函数了：`castersMap[typeof(LuaFunction)] = getLuaFunction;`

=== "Step 4"

    ```cs title="ObjectCasters.cs:408"
    private object getLuaFunction(RealStatePtr L, int idx, object target)
    {
        // 分支 A: 栈上是 userdata（被 Lua 包装过的 C# 委托）
        if (LuaAPI.lua_type(L, idx) == LuaTypes.LUA_TUSERDATA)
        {
            object obj = translator.SafeGetCSObj(L, idx);
            return (obj != null && obj is LuaFunction) ? obj : null;
        }

        // 分支 B: 不是函数 → 直接返回 null
        if (!LuaAPI.lua_isfunction(L, idx))
        {
            return null;
        }

        // 分支 C: 栈上确实是 Lua function，做"包装"
        LuaAPI.lua_pushvalue(L, idx); // 复制一份到栈顶
        return new LuaFunction(LuaAPI.luaL_ref(L), translator.luaEnv); // 入注册表
    }
    ```

    !!! question "为什么要复制一份？"
        因为下一步 `luaL_ref` 会把栈顶的值**弹掉**并存入注册表。如果不先复制，`idx = -1` 处的原值会被消掉，导致最终 lua_settop 还原栈时数量不对。
        其实这里因为无论如何都会 lua_settop(L, oldTop)，少不少一个不影响。但 getLuaFunction 是通用 caster，idx 可能是任何位置，所以保险起见统一先复制。

    !!! note "方法存入注册表并返回 C# 整数句柄"
        `LuaAPI.luaL_ref(L)` 将栈顶的值取下来，将它存入 Lua 注册表，然后返回一个整数句柄 ref，下次如果要获取这个值，通过 `lua_getref(L, ref)` 就能将这个值压入栈。

    !!! question "这种情况下注册表中的值如何才能正确被释放？"
        由于注册表是 Lua 的根对象，一直存在这个表中的值，无法被 Lua GC 释放。我们在 LuaFunction 被 Dispose 时，xLua 会调 `luaL_unref` 把这个 ref 释放，让 Lua GC 可以回收。

=== "Step 5"

    ```cs title=""
    return new LuaFunction(LuaAPI.luaL_ref(L), translator.luaEnv);
    ```

    LuaFunction 继承自 LuaBase，构造函数只是把 ref 和 luaEnv 存下来：

    ```cs title=""
    public LuaBase(int reference, LuaEnv luaenv)
    {
        luaReference = reference; // 注册表里的整数门牌号
        luaEnv = luaenv; // Lua 虚拟机环境
    }
    ```

    !!! note "之后的使用"
        之后 C# 任何时候要调用这个 Lua 函数：

        1. lua_getref(L, luaReference) 把函数从注册表拿回栈上
        2. 压参数
        3. lua_pcall 调用
        4. 从栈上取返回值

## 2. 执行 LuaFunction

LuaFunction 有两种执行方式：

- `LuaFunction.Call(params object[])`：装箱版本，性能差，不推荐。
- `LuaFunction.Action<T> / Func<T,TR>`：泛型版本，性能较好。

装修版本一般不推荐使用，这里不赘述。因为前面对 C# 跨域调用 Lua 已经有详细讲解了，所以这里简单讲讲泛型版本的大致步骤：

1. `lua_gettop` 记录栈顶锚点，`load_error_func` 把 trackback 压栈
2. `lua_getref(L, luaReference)` 将目标 Lua 函数从注册表拿出来压栈
3. `translator.PushByType<T>(L, arg)` 将参数入栈（这里如果是基础类型，走快路径执行就很快，并且没有类型转换）
4. `lua_pcall` 让 xlua.dll 执行栈上的函数，并消费栈上的参数，如果有返回值直接出栈，在 C# 这边直接得到
5. `lua_settop(L, oldTop)` 把栈恢复到调用前的高度，避免栈泄漏