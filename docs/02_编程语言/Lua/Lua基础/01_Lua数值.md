## 一、十六进制数的表示

``` Lua
--注意p代表2的幂，带p的这种格式的可读性很差，但是可以保留所有浮点数的精度，并且转换成十六进制比十进制更快

--0x0.2 = 2 / 16 = 0.125
print(0x0.2)

--0x1p-1 = 1 x 2 ^ -1 = 0.5
print(0x1p-1)

--0xa.bp2 = (10 + 11 / 16) x 2 ^ 2 = 42.75
print(0xa.bp2)

--将十进制数按十六进制格式化输出
print(string.format("%a", 419))
```

## 二、Lua5.3引入了整数类型integer

``` lua
--type统一认为是number（数值类型）
print("type:", type(3.0))    --number
print("type:", type(3))      --number

--math.type会区分integer和float（Lua5.3引入，注意对于Lua来说float表示的是双精度）
print("math.type:", math.type(3.0))    --integer
print("math.type:", math.type(3))      --float
```

## 三、Lua5.3引入的新运算符：//（floor除法）

``` lua
--结果是-5（-9 / 2 = -4.5，向下取整-5）
print(-9 // 2)

--这俩等效
print(100 % 32)
print(100 - (100 // 32) * 32)
```

## 四、两种取模运算的细节

``` lua
--Lua中取模运算都是先计算商然后通过商得到余数，%和math.fmod这俩的区别在于对商的计算不同

-- 商 = math.floor(-2 / 3) = -1，所以余数 = -2 - (-1 * 3) = 1
print(-2 % 3)
-- 商 = (-2 / 3)向0取整（注意不是向上也不是向下）= 0，所以余数 = -2
print(math.fmod(-2, 3))

--容易混淆的一个库函数math.modf，它的作用是将一个浮点数分为整数部分和小数部分
--注意得到的值符号相同，一定是整数部分 + 小数部分 = 原来的数
print(math.modf(-3.3))    --输出-3    -0.3
```

## 五、Lua数值类型的表示范围

``` lua
--Lua中不管是整型还是浮点数都用64个bit来存储（所以Lua的浮点数也是双精度，但是类型是float），需要注意的是整型有回环特征
math.maxinteger + 1 == math.mininteger    -->true
```

## 六、一些常用的计算技巧

``` lua
---保留到小数点后第几位
function Percision()
    local x = math.pi
    print(x - x % 0.01)    --3.14
    print(x - x % 0.001)   --3.141
end

---四舍五入
function NearestInteger()
    --math.floor(x + 0.5)
    print(math.floor(0.4 + 0.5))   --0
    print(math.floor(0.5 + 0.5))   --1
end
```