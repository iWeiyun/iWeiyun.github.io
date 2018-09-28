---
title: Swift is like Haskell
date: 2018-09-27 21:08:53
author: matthew
tags: haskell,swift,函数式
---



### 前言



Swift是一门多泛式语言，而且参考了很多其它语言的实现，所以总能在不同语言里看到一些Swift的影子。

前段时间看到一篇文章，[Swift is like Scala](<https://leverich.github.io/swiftislikescala/>)，里面做了Swift和Scala一些语法的对比，有些代码块语法上是很像的。最近又看到了[Swift is like Kotlin](<http://nilhcem.com/swift-is-like-kotlin/>)，[Swift is like Go](<http://repo.tiye.me/jiyinyiyong/swift-is-like-go/>)。感觉这些挺有趣的，最近刚好有了解一点Haskell，所以就有了这个想法来对比下相似点。

这儿只是列出两门语言一些类似的点，或语法，或概念上的。但在真正使用的时候，差别还是巨大的。如果想了解真正的工程中，Haskell的使用，可以参考下 [Github Haskell Star排名](https://github.com/trending/haskell?since=weekly)等



<!-- more -->



### 对比

----



##### Hello World

​	<font color=gray size=2>*Swift*</font>

```swift
print("Hello, world!")
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
putStrLn "hello, world!"
```



----



##### 变量定义

​	<font color=gray size=2>*Swift*</font>

```swift
let a = 1
var m = 2
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
-- haskell中没有var定义，只能定义不可修改的变量
let a = 1
```



---



##### 显示指定类型

​	<font color=gray size=2>*Swift*</font>

```swift
let a: Float = 5
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
let a = 5 :: Float
```



---



##### 字符串连接

​	<font color=gray size=2>*Swift*</font>

```swift
let ret = "111" + "222"
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
let ret = "111" ++ "222"
```



---



##### 数组定义

​	<font color=gray size=2>*Swift*</font>

```swift
let lst = ["1", "2", "3"]
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
let lst = ["1", "2", "3"]
```



---



##### 区间

​	<font color=gray size=2>*Swift*</font>

```swift
let lst = [1 ... 5]
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
let lst = [1 .. 5]	-- [1, 2, 3, 4, 5]
```



------



##### 函数定义

​	<font color=gray size=2>*Swift*</font>

```swift
func myAdd(_ a: Int, _ b: Int) -> Int { return a + b }
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
myAdd a b = a + b
```



---



##### 泛型函数

​	<font color=gray size=2>*Swift*</font>

```swift
func myAdd<T: Numeric>(_ a: T, _ b: T) -> T { return a + b }
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
myAdd a b = a + b 
```



------



##### 返回元组

​	<font color=gray size=2>*Swift*</font>

```swift
func process(_ a: Int, _ b: Int) -> (Int, Int) {
    return (a + b, a * b)
}
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
process a b = (a + b, a * b)
```



---



##### 操作符另外的调用方式

​	<font color=gray size=2>*Swift*</font>

```swift
(+)(1, 2)	// 输出3
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
(+) 1 2
```



------



##### 自定义操作符

​	<font color=gray size=2>*Swift*</font>

```swift
infix operator <^> : AdditionPrecedence
func <^>(lhs: Int, rhs: Int) -> Int { return lhs + rhs }
5 <^> 6		// 11
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
(<^>) :: Int -> Int -> Int
a <^> b = a + b
5 <^> 6		-- 11
```



------



##### Map

​	<font color=gray size=2>*Swift*</font>

```swift
[1, 2, 3].map { $0 * 2}	// [2, 4, 6]
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
map (*2) [1, 2, 3]		-- [2, 4, 6]
```



------



##### 数据类型

​	<font color=gray size=2>*Swift*</font>

```swift
public struct CGPoint {
    public var x: CGFloat
    public var y: CGFloat
}
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
data Point = Point Double Double
              deriving (Show)
```



---



##### 枚举类型

​	<font color=gray size=2>*Swift*</font>

```swift
enum Shape {
    case circle(CGPoint, CGFloat)
    case rectangle(CGPoint, CGFloat, CGFloat)
}
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
data Shape = Circle Point Double
           | Rectangle Point Double Double
             deriving (Show)
```



---



##### 模式匹配

​	<font color=gray size=2>*Swift*</font>

```swift
let e = Shape.circle(CGPoint(x: 0, y: 0), 100)
switch e {
case let .circle(pt, _):
    // code
case let .rectangle(pt, _, _):
    // code
}
```

​	<font color=gray size=2>*Haskell*</font>

```haskell
shapeInfo :: Shape -> String
shapeInfo (Circle pt _) = "Circle point" ++ show pt
shapeInfo (Rectangle pt _ _) = "Rectangle point" ++ show pt
```



---



#### 参考资料

[Real World Haskell 中文版](http://cnhaskell.com/index.html)