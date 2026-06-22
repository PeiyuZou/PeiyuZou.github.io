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

### 2. 执行 LuaFunction

LuaFunction 有两种执行方式：

- `LuaFunction.Call(params object[])`：装箱版本，性能差，不推荐。
- `LuaFunction.Action<T> / Func<T,TR>`：泛型版本，性能较好。

装修版本一般不推荐使用，这里不赘述。因为前面对 C# 跨域调用 Lua 已经有详细讲解了，所以这里简单讲讲泛型版本的大致步骤：

1. `lua_gettop` 记录栈顶锚点，`load_error_func` 把 trackback 压栈
2. `lua_getref(L, luaReference)` 将目标 Lua 函数从注册表拿出来压栈
3. `translator.PushByType<T>(L, arg)` 将参数入栈（这里如果是基础类型，走快路径执行就很快，并且没有类型转换）
4. `lua_pcall` 让 xlua.dll 执行栈上的函数，并消费栈上的参数，如果有返回值直接出栈，在 C# 这边直接得到
5. `lua_settop(L, oldTop)` 把栈恢复到调用前的高度，避免栈泄漏

## 使用强类型委托（推荐）

强类型委托是比较推荐的一种方案，相较于泛型版本的 LuaFunction，它的灵活性更高。下面是一个使用示例：

```lua title="假设 lua 一侧有这样的函数定义"
local Foo = {}

-- 实现一个简单的加法计算
function Foo.Add(a, b)
    return a + b
end

return Foo
```

```cs title="C# 一侧通过强类型委托的方式实现跨域调用"
// 1. 声明并打标记
[CSharpCallLua]
public delegate int FooAddDelegate(int a, int b);

// 2. 拿委托
var addDelegate = fooTable.Get<FooAddDelegate>("Add");

// 3. 调用
int sum = addDelegate(1, 2);
```

下面逐步拆解：

### 1. 前面的调用路径和 Get<LuaFunction> 一样

前面的流程一致，都是调用 Get<T>，简要带过一下：

- `lua_gettop / lua_getref / PushByType("Add") / xlua_pgettable` 执行完，栈顶得到目标 **lua function**
- `translator.Get<FooAddDelegate>` 快表里没有 `FooAddDelegate`，走慢路径
- `GetObject(L, -1, typeof(CalcDelegate)` lua function 不是 userdata，所以进入 `GetCaster` 路径

### 2. 分歧出现，非预定义的 caster 有各种处理逻辑

```cs title=""
public ObjectCast GetCaster(Type type)
{
    if (type.IsByRef) type = type.GetElementType();

    Type underlyingType = Nullable.GetUnderlyingType(type);
    if (underlyingType != null)
    {
        return genNullableCaster(GetCaster(underlyingType));
    }
    ObjectCast oc;
    if (!castersMap.TryGetValue(type, out oc))
    {
        oc = genCaster(type);
        castersMap.Add(type, oc);
    }
    return oc;
}
```

`LuaFunction` 在 `casterMap` 中预定义了，而 `FooAddDelegate` 是自定义委托，所以不会有预定义，这里走 `genCaster` 分支。

### 3. 动态生成类型转换

```cs title=""
private ObjectCast genCaster(Type type)
{
    // 通用前置：先尝试当 userdata 取（兼容 C# 委托被反包装的场景）
    ObjectCast fixTypeGetter = (L, idx, target) =>
    {
        if (LuaAPI.lua_type(L, idx) == LuaTypes.LUA_TUSERDATA)
        {
            object obj = translator.SafeGetCSObj(L, idx);
            return (obj != null && type.IsAssignableFrom(obj.GetType())) ? obj : null;
        }
        return null;
    };

    if (typeof(Delegate).IsAssignableFrom(type))   // FooAddDelegate 满足这个条件
    {
        return (L, idx, target) =>
        {
            object obj = fixTypeGetter(L, idx, target);
            if (obj != null) return obj;          // 尝试当 userdata 取

            if (!LuaAPI.lua_isfunction(L, idx))
                return null;

            return translator.CreateDelegateBridge(L, type, idx);  // 核心入口
        };
    }
    else if (type.IsInterface()) { ... }
    else if (type.IsEnum())      { ... }
    else if (type.IsArray)       { ... }
    ...
}
```

由于 `FooAddDelegate` 是委托，走委托的这个分支。注意这里返回一个 lambda 式子，返回后会缓存到 `casterMap` 中，所以动态生成只发生一次。

### 4. CreateDelegateBridge：核心逻辑

这里逻辑稍微复杂了一些，要搞清楚发生了什么，需要先建立一些心智模型：

#### 4.1 委托的本质

C# 委托 Delegate 主要包含两个成员：

- Method：委托保存的方法指针
- Target：这个方法所属的对象

所以，委托可以简单理解为保存了 `Target.Method`，发生调用时等价于 `Target.Method()`，这里不讲过深。

#### 4.2 关系图

![关系图](xLua_13.png)
/// caption
关系图
///

一个 lua function 需要在 C# 侧有一个 bridge 对象与之绑定，这个 bridge 对象还需要定义一个操作栈的函数，确保匹配这个 lua function 的函数签名，以便正确地操作虚拟机栈，传递正确的参数和返回值。所以对于 C# 侧来说，一个 bridge 对象可以看做是对一个 lua function 的代理，它们是一一对应的。

`DelegateBridge` 类本身不定义类似 `__Gen_Delegate_Imp_5` 这些桥接方法，因为 xLua 无法主动知道哪些 lua function 会被 C# 侧以委托的方式调用。xLua 提供了两种方式来解决这个问题，第一种是用户主动给自定义委托打 `[CSharpCallLua]` 标签，这样可以在 Wrap 生成时自动生成对应的桥接方法；第二种方式是通过动态 Emit 在代码执行时直接生成对应的 IL 内容到内存中，这种方式不会落地 cs 文件，并且只能在使用 .NET Framework 的编辑器环境下生效，也许是为了便于开发期使用。

对于 C# 上层实际的单个委托来说，xLua 的工作就是先找到目标 lua 函数对应的 bridge 对象，然后绑定给这个委托对象，确保能被正确调用；另外，还需要建立缓存，以便在下次别的地方获取同一个 lua 函数时可以直接命中。

#### 4.3 讲解函数执行步骤

=== "Step 1"

    ```cs title="先查这个 lua 函数是否已经有 bridge"
    // 将栈顶原有的 function 复制一份到栈顶
    LuaAPI.lua_pushvalue(L, idx);

    // REGISTRY[function]，后续步骤可以看到：
    // 首次建立 bridge 时做了双向映射，为的就是这里可以反查
    LuaAPI.lua_rawget(L, LuaIndexes.LUA_REGISTRYINDEX);

    if (!LuaAPI.lua_isnil(L, -1)) // 已经存在对应的 bridge
    {
        int referenced = LuaAPI.xlua_tointeger(L, -1);
        LuaAPI.lua_pop(L, 1); // 取出 ref 后，栈顶的 ref 没用了弹出

        // 这里 delegate_bridges 使用的弱引用，为了被正确 GC
        // bridge 还在被引用就复用，没有就走到首次建立的分支
        if (delegate_bridges[referenced].IsAlive)
        {
            if (delegateType == null)
            {
                return delegate_bridges[referenced].Target;
            }
            DelegateBridgeBase exist_bridge
                = delegate_bridges[referenced].Target as DelegateBridgeBase;
            Delegate exist_delegate;
            if (exist_bridge.TryGetDelegate(delegateType, out exist_delegate))
            {
                // bridge 还包含这个类型的委托，直接复用
                return exist_delegate;
            }
            else
            {
                // bridge 还有效，但这个类型的委托曾被回收过，再建立一次
                exist_delegate = getDelegate(exist_bridge, delegateType);
                // 加入 bridge 的内部缓存
                exist_bridge.AddDelegate(delegateType, exist_delegate);
                return exist_delegate;
            }
        }
    }
    else
    {
        LuaAPI.lua_pop(L, 1); // 不存在对应的 bridge，
        // 在首次建立 bridge 之前先把栈顶的 ref 弹出
    }
    ```

=== "Step 2"

    ```cs title="首次建立 bridge 先做双向映射"
    // 复制 function 到栈顶
    LuaAPI.lua_pushvalue(L, idx);
    // REGISTRY[ref] = function (弹掉了栈顶 function)
    int reference = LuaAPI.luaL_ref(L);
    // 再复制一份 function
    LuaAPI.lua_pushvalue(L, idx);
    // 把 ref 数字也压栈
    LuaAPI.lua_pushnumber(L, reference);
    // REGISTRY[function] = ref（弹掉这两个）
    LuaAPI.lua_rawset(L, LuaIndexes.LUA_REGISTRYINDEX);
    ```

=== "Step 3"

    ```cs title="创建 DelegateBridge 对象"
    #if (UNITY_EDITOR || XLUA_GENERAL) && !NET_STANDARD_2_0
        if (!DelegateBridge.Gen_Flag)
        {
            // 编辑器 + .NET Framework 环境下，如果没有走 Wrap Gen
            // 则会走到这里，这里使用反射创建，而没有像下面这个分支
            // 直接 new DelegateBridge，后续会讲为什么
            bridge = Activator.CreateInstance(delegate_birdge_type,
                new object[] { reference, luaEnv }) as DelegateBridgeBase;
        }
        else
    #endif
        {
            bridge = new DelegateBridge(reference, luaEnv);
        }
    ```

=== "Step 4"

    ```cs title="生成真正的委托对象，并交给 bridge 缓存住"
    // 这一步创建并返回真正的委托对象，后续会讲解其内容
    var ret = getDelegate(bridge, delegateType);
    // 加入 bridge 缓存，下次有同类型委托要获取，直接复用
    bridge.AddDelegate(delegateType, ret);
    // 使用弱引用，确保被正确 GC
    delegate_bridges[reference] = new WeakReference(bridge);
    return ret;
    ```

#### 4.4 为什么使用弱引用缓存 bridge ?

如果不使用弱引用的话，一个 bridge 会被两个地方引用，一个是上层的委托对象，比如示例中的 `m_FooAdd`，第二个是 `delegate_bridges` 自己。一旦 `m_FooAdd` 被清理掉，`delegate_bridges` 长期持有这个 bridge 会导致 GC 无法正确回收它，甚至连 lua 那份 function 也无法被 lua 回收。

所以使用弱引用是为了 bridge 被正确回收，当 GC 发现 bridge 没有被引用时，会回收它，同时触发它的析构函数，我们来看下它的析构函数（派生自 LuaBase，所以这里是 LuaBase 的析构函数）：

```cs title="LuaBase析构函数"
~LuaBase()
{
    Dispose(false);
}

public virtual void Dispose(bool disposeManagedResources)
{
    if (!disposed)
    {
        if (luaReference != 0)
        {
#if THREAD_SAFE || HOTFIX_ENABLE
            lock (luaEnv.luaEnvLock)
            {
#endif
                bool is_delegate = this is DelegateBridgeBase;
                if (disposeManagedResources)
                {
                    luaEnv.translator.ReleaseLuaBase(luaEnv.L, luaReference, is_delegate);
                }
                else //will dispse by LuaEnv.GC
                {
                    // 传入 false 走这个分支
                    luaEnv.equeueGCAction(new LuaEnv.GCAction { Reference = luaReference, IsDelegate = is_delegate });
                }
#if THREAD_SAFE || HOTFIX_ENABLE
            }
#endif
        }
        disposed = true;
    }
}
```

这里会入队一个 Lua GC 的命令，最终走到 `translator.ReleaseLuaBase(_L, gca.Reference, gca.IsDelegate);`

```cs title="释放一个LuaBase"
public void ReleaseLuaBase(RealStatePtr L, int reference, bool is_delegate)
{
    if(is_delegate)
    {
        // 1. 做一些类型检查
        LuaAPI.xlua_rawgeti(L, LuaIndexes.LUA_REGISTRYINDEX, reference);
        if (LuaAPI.lua_isnil(L, -1))
        {
            LuaAPI.lua_pop(L, 1);
        }
        else
        {
            LuaAPI.lua_pushvalue(L, -1);
            LuaAPI.lua_rawget(L, LuaIndexes.LUA_REGISTRYINDEX);
            if (LuaAPI.lua_type(L, -1) == LuaTypes.LUA_TNUMBER && LuaAPI.xlua_tointeger(L, -1) == reference) //
            {
                //UnityEngine.Debug.LogWarning("release delegate ref = " + luaReference);
                LuaAPI.lua_pop(L, 1);// pop LUA_REGISTRYINDEX[func]
                LuaAPI.lua_pushnil(L);
                LuaAPI.lua_rawset(L, LuaIndexes.LUA_REGISTRYINDEX); // LUA_REGISTRYINDEX[func] = nil
            }
            else //another Delegate ref the function before the GC tick
            {
                LuaAPI.lua_pop(L, 2); // pop LUA_REGISTRYINDEX[func] & func
            }
        }

        // Lua 从 REGISTER 解绑对应的 function，确保 lua GC 正确工作
        LuaAPI.lua_unref(L, reference);
        // bridge 弱引用没用了，移除掉
        delegate_bridges.Remove(reference);
    }
    else
    {
        // 其他类型的对象，只需解绑，等待 GC 即可
        LuaAPI.lua_unref(L, reference);
    }
}
```

这就是为什么使用弱引用的原因，如果不这样设计，C# 和 Lua 两边的无效引用都无法被回收。

#### 4.5 为什么 .NET Framework 会走反射创建 bridge ?

使用 .NET Framework 的编辑器环境，在没有生成 wraps 代码文件的时候，为了开发期方便，会采用动态 Emit 的方式，直接在内存中生成对应桥接方法的 IL 内容。

怎么理解 **Emit** ？可以这样说：反射是运行时“看和调用已有代码”；而 Emit 是运行时“制造新代码”。

可以理解为运行时生成了下面这种类的等效内容：

``` cs title="动态 Emit 的等效内容"
class XLuaGenDelegateImpl0 : DelegateBridge
{
    public override Delegate GetDelegateByType(Type type)
    {
        ...
    }
}
```

这样就算没有生成 xLua Wraps，Editor 下也能跑，开发体验比较好。所以在创建 bridge 时，不得不使用反射的方式。

在生成 wraps 的情况下，并没有从 `DelegateBridge` 派生，而是生成了局部类的新内容，相当于给 `DelegateBridge` 添加了新函数，所以可以直接 `new DelegateBridge`。

那为什么 .NET Standard 不走动态 Emit ？准确来说，.NET Standard 不一定能正常地动态 Emit 并执行生成代码。一方面，它是一个兼容性规范，目的是让代码能跑在更多 runtime 上，而不是保证完整 .NET Framework 行为。另外一方面，Unity 的 .NET Standard Profile 对 Reflection.Emit 支持不稳定或不完整。

### 5. getDelegate 生成真正的委托对象

```cs title=""
Delegate getDelegate(DelegateBridgeBase bridge, Type delegateType)
{
    // 第一种，最常见的情况，Wraps 或 Emit 直接 override 了这个函数
    // bridge 这个函数直接创建了一个委托并返回
    Delegate ret = bridge.GetDelegateByType(delegateType);

    if (ret != null)
    {
        return ret;
    }

    if (delegateType == typeof(Delegate) || delegateType == typeof(MulticastDelegate))
    {
        return null;
    }

    Func<DelegateBridgeBase, Delegate> delegateCreator;
    if (!delegateCreatorCache.TryGetValue(delegateType, out delegateCreator))
    {
        // 第二种，如果没有生成对应的方法
        // 尝试在 bridge 上找一个签名匹配的方法，并包装为一个委托对象
        MethodInfo delegateMethod = delegateType.GetMethod("Invoke");
        var methods = bridge.GetType().GetMethods(BindingFlags.Public | BindingFlags.Instance | BindingFlags.DeclaredOnly).Where(m => !m.IsGenericMethodDefinition && (m.Name.StartsWith("__Gen_Delegate_Imp") || m.Name == "Action")).ToArray();
        for (int i = 0; i < methods.Length; i++)
        {
            if (!methods[i].IsConstructor && Utils.IsParamsMatch(delegateMethod, methods[i]))
            {
                var foundMethod = methods[i];
                delegateCreator = (o) =>
#if !UNITY_WSA || UNITY_EDITOR
                    Delegate.CreateDelegate(delegateType, o, foundMethod);
#else
                    foundMethod.CreateDelegate(delegateType, o);
#endif
                break;
            }
        }

        // 第三种情况，回退到泛型 Action<...>/Func<...>
        if (delegateCreator == null)
        {
            delegateCreator = getCreatorUsingGeneric(bridge, delegateType, delegateMethod);
        }
        delegateCreatorCache.Add(delegateType, delegateCreator);
    }

    ret = delegateCreator(bridge);
    if (ret != null)
    {
        return ret;
    }

    throw new InvalidCastException("This type must add to CSharpCallLua: " + delegateType.GetFriendlyName());
}
```

正常来讲都是走第一种路径，Wraps 生成的时候会重写 `bridge.GetDelegateByType(delegateType)` 函数，内容大致如下：

```cs title=""
public override Delegate GetDelegateByType(Type type)
{
    if (type == typeof(FooAddDelegate))
    {
        return new FooAddDelegate(__Gen_Delegate_Imp5);
    }

    if (type == ...)
    ...
}
```

### 6. 总结

强类型委托的整个调用链和设计相对较复杂，但性能比较好，它的运作原理可以归纳为以下几点：

- bridge 是核心，通过它绑定一个 lua function，并负责定义桥接函数来操作 Lua 虚拟机栈完成调用
- 具体包含的桥接函数是通过 Wraps Gen 或者编辑器环境下动态 Emit 生成的
- 返回给上层的委托对象，在构建时绑定为 bridge 的某某桥接函数，这样实际执行时就是调用 bridge 的某个桥接函数
- 注意弱引用的设计，是为了两边的域能正确完成回收