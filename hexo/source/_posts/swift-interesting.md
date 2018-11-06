---
title: 你不知道的Swift
date: 2018-11-06 22:49:17
author: matthew
tags: swift
---

​	


---

​	



### 自定义运算符不仅限于Ascii符号

```swift
infix operator ❤ : MultiplicationPrecedence
func ❤(lhs: Int, rhs: Int) -> Int { return lhs + rhs }
r = 1 ❤ 5	// 结果是6
```

除了❤，类似☢、☂、☯等都可以用，而且这些符号都是可以组合起来的

​	



------

​	



### 空集合，是无法判断出有效类型的

```swift
let arr = [Int]()
if arr is [String] {
    print("Match!!")	// 会输出Match!!
}
```

​	



------

​	



### 可变参数，在函数内部就是数组

```swift
func test(params: String...) {
    let t = type(of: params)
    print(t)	// 输出：Array<String>
    print(params)	// 输出：["1", "2"]
}
test(params: "1", "2")
```

​	



------

​	



### Optional类型，也是有map方法的

```swift
let i: Int? = 10
let r = i.map { $0 + 1 }
print(r)	// Optional(11)
```

有些场景下可以极大简化代码

​	



------

​	



### print是有更多参数可用的

```swift
print("Hello", "World", separator: " ")	// 输出Hello World，中间用空格分割
```

```swift
// 下面两句输出： HelloWorld，不会换行
print("Hello" terminator: "")
print("World")
```

除此之外还有debugPrint，可能会输出更详细的信息

​	



------

​	



### ~= 可以很方便判断值是否在区间内

```swift
let r = 1...5 ~= 3
print(r)	// 输出：true
```

​	



------

​	



### switch判断，可以更强大

case后面跟可以不同类型，但需要重载 ~= 运算符

```swift
struct Person {
    let name : String
}

func ~=(pattern: String, value: Person) -> Bool {
    return value.name == pattern
}

let person = Person(name: "Alessandro")
switch person {
case "Alessandro":
    print("Hey it's me!")
default:
    print("Not me")
}
```

​	



------

​	



### 空元组，可以在不关心类型及数据时使用

```swift
func emptyTuple(_: ()) {
    // some code
}
emptyTuple(())	// 调用时也需要传入空元组
```

在Rx中，是有大量这样的使用，比如事件通知，不需要传数据，只要触发一下的情况，可能会用()

​	



Swift中，Void也是用的空元组来定义的

```swift
public typealias Void = ()
```

​	



------

​	



### 一起遍历两个Sequence时，可以用zip

```swift
let lst1 = [1, 2, 3]
let lst2 = ["a", "b", "c", "d"]
for (a, b) in zip(lst1, lst2) {
    print("\(a)\(b)")	// 输出1a 2b 3c，以短的列表为准
}
```

​	



------

​	



### 全局变量默认就是lazy的

```swift
func globalFunc() -> Int {
    print("Hello")
    return 42
}
var globalVar = globalFunc()	// 只有globalVar被使用时，globalFunc才会被调用
```

苹果源码仓库里有相应的测试代码，可以详细看到什么情况下会是懒加载：[lazy_properties.swift](https://github.com/apple/swift/blob/master/test/decl/var/lazy_properties.swift)

​	



------

​	



### precondition，比assert更严格的检查

用法上与assert类似，但更严格，只有-Ounchecked选项才能关闭，但这样会很危险

```swift
precondition(x >= 0)
```

​	



------

​	



### struct在没有自定义init时，系统会帮我们生成

```swift
struct MySize {
    var width: CGFloat = 0
    var height: CGFloat = 0
}
let s1 = MySize(width: 1, height: 1)
let s2 = MySize()	// 如果上面没有指定默认值，则不会生成这个初始化方法
```

​	



------

​	



### compactMap可以从列表中筛选出指定类型

```swift
let arrs: [Any] = [1, 2, "3", "4", 5]
let r = arrs.compactMap { $0 as? String }	// r已经被推断为[String]类型
print(r)	// 输出：["3", "4"]
```

​	



------

​	



### 从Dictionary获取值时，可以提供默认值

```swift
let dict = ["1": 1]
let o = dict["2", default: 2]
print(o)	// 输出：2
```

​	



------

​	



### 运算符也是方法

```swift
let r = (+)(5, 6)
print(r)	// 输出：11
```

​	

----

