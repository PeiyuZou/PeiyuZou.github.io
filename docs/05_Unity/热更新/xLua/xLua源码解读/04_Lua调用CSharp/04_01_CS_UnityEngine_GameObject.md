本章开始解析 Lua 调用 C# 的流程，由于 C# 调用 Lua 的流程讲的比较细，而且对整个 xLua 的运作设计有一个大体说明，所以这章不会讲的那么细，仅指出关键节点和设计。

Lua 调用 C# 的前提是 C# 类型必须被导出，通过两种标记声明导出范围：

- [LuaCallCSharp]：编译期生成 XxxWrap
- [CSharpCallLua]：编译期为该委托/接口生成 DelegateBridge / InterfaceBridge

这一篇来看一种场景的调用场景：`local go = CS.UnityEngine.GameObject("name")`

## CS 全局表

CS 全局表在 C# LuaEnv 构造时对应地构建起来：

```cs title="LuaEnv构造函数"
public LuaEnv()
{
    ...
    DoString(init_xlua, "Init");
    init_xlua = null;
    ...
}

private string init_xlua = @"......" // 一大串lua代码
```

```lua title="init_xlua的全部内容"
local metatable = {}
local rawget = rawget
local setmetatable = setmetatable
local import_type = xlua.import_type
local import_generic_type = xlua.import_generic_type
local load_assembly = xlua.load_assembly

function metatable:__index(key)
    local fqn = rawget(self,'.fqn')
    fqn = ((fqn and fqn .. '.') or '') .. key

    local obj = import_type(fqn)

    if obj == nil then
        -- It might be an assembly, so we load it too.
        obj = { ['.fqn'] = fqn }
        setmetatable(obj, metatable)
    elseif obj == true then
        return rawget(self, key)
    end

    -- Cache this lookup
    rawset(self, key, obj)
    return obj
end

function metatable:__newindex()
    error('No such type: ' .. rawget(self,'.fqn'), 2)
end

-- A non-type has been called; e.g. foo = System.Foo()
function metatable:__call(...)
    local n = select('#', ...)
    local fqn = rawget(self,'.fqn')
    if n > 0 then
        local gt = import_generic_type(fqn, ...)
        if gt then
            return rawget(CS, gt)
        end
    end
    error('No such type: ' .. fqn, 2)
end

CS = CS or {}
setmetatable(CS, metatable)

typeof = function(t) return t.UnderlyingSystemType end
cast = xlua.cast
if not setfenv or not getfenv then
    local function getfunction(level)
        local info = debug.getinfo(level + 1, 'f')
        return info and info.func
    end

    function setfenv(fn, env)
        if type(fn) == 'number' then fn = getfunction(fn + 1) end
        local i = 1
        while true do
        local name = debug.getupvalue(fn, i)
        if name == '_ENV' then
            debug.upvaluejoin(fn, i, (function()
            return env
            end), 1)
            break
        elseif not name then
            break
        end

        i = i + 1
        end

        return fn
    end

    function getfenv(fn)
        if type(fn) == 'number' then fn = getfunction(fn + 1) end
        local i = 1
        while true do
        local name, val = debug.getupvalue(fn, i)
        if name == '_ENV' then
            return val
        elseif not name then
            break
        end
        i = i + 1
        end
    end
end

xlua.hotfix = function(cs, field, func)
    if func == nil then func = false end
    local tbl = (type(field) == 'table') and field or {[field] = func}
    for k, v in pairs(tbl) do
        local cflag = ''
        if k == '.ctor' then
            cflag = '_c'
            k = 'ctor'
        end
        local f = type(v) == 'function' and v or nil
        xlua.access(cs, cflag .. '__Hotfix0_'..k, f) -- at least one
        pcall(function()
            for i = 1, 99 do
                xlua.access(cs, cflag .. '__Hotfix'..i..'_'..k, f)
            end
        end)
    end
    xlua.private_accessible(cs)
end
xlua.getmetatable = function(cs)
    return xlua.metatable_operation(cs)
end
xlua.setmetatable = function(cs, mt)
    return xlua.metatable_operation(cs, mt)
end
xlua.setclass = function(parent, name, impl)
    impl.UnderlyingSystemType = parent[name].UnderlyingSystemType
    rawset(parent, name, impl)
end

local base_mt = {
    __index = function(t, k)
        local csobj = t['__csobj']
        local func = csobj['<>xLuaBaseProxy_'..k]
        return function(_, ...)
                return func(csobj, ...)
        end
    end
}
base = function(csobj)
    return setmetatable({__csobj = csobj}, base_mt)
end
```

这一步把 CS 全局表注册起来，并给它设置元表扩展它的行为。

## 链式调用

形如 `CS.AAA.BBB.CCC` 这样的链式调用有个问题要解决，就是如何知道谁是真正的类型而不是命名空间，CS 元表以 `.fqn`（全称fully qualified name，全限定名称）这个键解决这个问题。当处理 `CS.AAA` 时，实际触发 CS 元表的 `__index` 函数。

```lua title="init_xlua"
function metatable:__index(key)
    local fqn = rawget(self,'.fqn')
    fqn = ((fqn and fqn..'.') or '')..key
    local obj = import_type(fqn) -- 这一步会进入 C#
    if obj == nil then
        obj = { ['.fqn'] = fqn } -- 命名空间占位表
        setmetatable(obj, metatable)
    elseif obj == true then
        return rawget(self, key)
    end
    rawset(self, key, obj) -- 命名空间占位表写入
    return obj
end
```

这里核心关键是 `import_type` 函数，它指向 C# 侧的 `StaticLuaCallbacks.ImportType(RealStatePtr L)` 函数：

```cs title="StaticLuaCallbacks.cs"
[MonoPInvokeCallback(typeof(LuaCSFunction))]
public static int ImportType(RealStatePtr L)
{
    try
    {
        ObjectTranslator translator = ObjectTranslatorPool.Instance.Find(L);
        string className = LuaAPI.lua_tostring(L, 1);
        Type type = translator.FindType(className);
        if (type != null)
        {
            if (translator.GetTypeId(L, type) >= 0)
            {
                LuaAPI.lua_pushboolean(L, true);
            }
            else
            {
                return LuaAPI.luaL_error(L, "can not load type " + type);
            }
        }
        else
        {
            LuaAPI.lua_pushnil(L);
        }
        return 1;
    }
    catch (System.Exception e)
    {
        return LuaAPI.luaL_error(L, "c# exception in xlua.import_type:" + e);
    }
}
```

这里实际触发的就是 `translator.GetTypeId` 注册类型元表，所以当这里传入命名空间时，不是类型无法成功注册对应元表，所以栈顶会压入一个 `nil`，所以 CS 元表那边得到的返回值是一个 nil。这里继续构建一个 `.fqn` 的占位表作为命名空间表并缓存，以便接下来能顺利进行链式调用。

继续走，如果 `CCC` 才是真正的类型，那么 `CS.AAA.BBB.CCC` 的时候 `import_type` 返回的就是 true 了。这里由于 `translator.GetTypeId` 触发了 `xxxWrap.__Register(L)` 已经将类型元表注册好了，所以直接 `return rawget(self, key)` 返回 `CCC` 类型元表。

## 执行构造函数

`CS.UnityEngine.GameObject("name")` 这一句实际调用的是 `__call` 键，对应 `UnityEngineGameObjectWrap.__CreateInstance(L)` 函数：

```cs title="UnityEngine_GameObjectWrap.cs"
static int __CreateInstance(RealStatePtr L) {
    int paramCount = LuaAPI.lua_gettop(L);
    if (paramCount == 1) {                 // 无参构造（self 算一个参数）
        var go = new GameObject();
        translator.PushObject(L, go, GameObjectTypeId);
        return 1;
    }
    if (paramCount == 2 && LuaAPI.lua_type(L,2) == LUA_TSTRING) {
        string name = LuaAPI.lua_tostring(L, 2);
        var go = new GameObject(name);
        translator.PushObject(L, go, GameObjectTypeId);
        return 1;
    }
    // ... 其它重载分支
    return LuaAPI.luaL_error(L, "invalid arguments to GameObject ctor");
}
```

`translator.PushObject` 把刚 new 出来的 C# GameObject 注册到 objects/reverseMap，并通过 xlua_pushcsobj 把一个 userdata 压回 Lua 栈。Lua 端 `local go = ...`，`go` 是一个带 metatable 的 userdata。后续 go:Method() 全部走该 metatable 的 __index 键。