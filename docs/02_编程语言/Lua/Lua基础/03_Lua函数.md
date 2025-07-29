## 一、可变长参数

用符号...表示

```lua
--表示用a和b存储...参数的前两个元素
local a, b = ...

--select函数
--第一个参数为selector，表示返回参数列表第n个参数后的所有元素
print(select(2, "a", "b", "c"))    --> b    c
--selector为字符串"#"时，表示返回参数个数
print(select("#", "a", "b", "c"))  --> 3
--可用于取可变长参数个数
select("#", ...)
```

## 二、正确的尾调用
形如下方：
```lua
function f(x)
    x = x + 1
    return g(x)
end
```

当函数最后一个动作是返回调用另外一个函数而没有再进行其他工作时，就形成了尾调用。在g返回时，程序的执行路径会直接返回到调用f的位置。所以在进行尾调用时，不使用任何额外的栈空间，所以一个程序中能嵌套的尾调用数量是无限的，这样调用永远不会发生栈溢出。

不是尾调用的情况：
```lua
--相当于 return nil，并没有尾调用
function f(x)
    g(x)
end

--[[
相当于:
local y = g(x)
return y + 1
]]
return g(x) + 1

return x or g(x)

return (g(x))
```