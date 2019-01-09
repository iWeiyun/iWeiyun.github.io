---
title: 微云swift编码规范
date: 2019-01-09 18:37:53
author: matthew
tags: swift,style
---



#### 类、函数等，左大括号不另一起行，并且跟前面留有空格

Good

```swift
func myFunc() {
}

class MyClass() {  
}
```

Bad

```swift
func myFunc() 
{
}
```

​	

#### 函数、类中间要空一行

Good

```swift
func myFunc1() {
}

func myFunc2() {
}

class MyClass1 {
}
```

Bad

```swift
func myFunc1() {
}
func myFunc2() {
}


class MyClass1 {
}
```

​	

#### 代码逻辑不同块之间，要空一行

Good

```swift
func process() {
    // do one thing
    doCode1()
    
    // do another thing
    doAnother()
}
```

Bad

```swift
func process() {
    // do one thing
    doCode1()
    // do another thing
    doAnother()
}
```

​	

#### 缩进为一个tab (4个空格的宽度)

​	

#### 空行里不能有空的tab、空格

​	

#### 二元运算符，前后都要有空格

Good

```swift
let i = 5 + 6
let r = i % 10
```

Bad

```swift
let i=5+6
let r=i%10
```

​	

#### 区间运算符两边也要有空格

Good

```swift
let range = 1 ..< 10
```

Bad

```swift
let range = 1..<10
```

​	

#### 逗号后面跟空格

Good

```swift
let arr = [1, 2, 3, 4]
```

Bad

```swift
let arr = [1,2,3,4]
```

​	

#### 注释符号，与注释内容之间加空格

Good

```swift
print("Hello")	// 打印Hello
```

Bad

```swift
print("Hello")//打印Hello
```

​	

#### 类继承、参数名和类型之间等，冒号前面不加空格，但后面跟空格

Good

```swift
class MyClass: NSObject {
}

func myFunc(value: Int) {
}
```

Bad

```swift
class MyClass : NSObject {
}

func myFunc(value:Int) {
}
```

​	

#### 自定义操作符，声明及实现，两边都要有空格隔开

Good

```swift
infix operator ||| : RxPrecedence
public func ||| <T, U>(lhs: T, rhs: T) -> U {
}
```

Bad

```swift
infix operator |||: RxPrecedence
public func |||<T, U>(lhs: T, rhs: T) -> U {
}
```

​	

#### if后面的else，跟着上一个if的右括号

Good

```swift
if flag {
    // code
} else {
    // code
}
```

Bad

```swift
if flag
{
    // code
}
else
{
    // code
}
```

​	

#### switch中，case跟switch左对齐

Good

```swift
switch value {
case 1:
    // code
case 2:
    // code
default:
    // code
}
```

Bad

```swift
switch value {
    case 1:
        // code
    case 2:
        // code
    default:
        // code
}
```

​	

#### 函数体长度不超过200行

​	

#### 单行不能超过200个字符

​	

#### 单类体长度不超过300行

​	

#### 实现每个协议时，在单独的extension里来实现

Good

```swift
class MyViewController: UIViewController {
}

extension MyViewController: UITableViewDataSource {
}

extension MyViewController: UIScrollViewDelegate {
}
```

Bad

```swift
class MyViewController: UIViewController, UITableViewDataSource, UIScrollViewDelegate {
}
```

​	

#### 闭包中的单表达式，省略return

Good

```swift
let r = arr.filter { $0 % 2 == 0 }
```

Bad

```swift
let r = arr.filter { return $0 % 2 == 0 }
```

​	

#### 简单闭包，写在同一行

Good

```swift
let r = arr.filter { $0 % 2 == 0 }
```

Bad

```swift
let r = arr.filter {
    $0 % 2 == 0
}
```

​	

#### 尾随闭包，在单闭包参数时才使用

Good

```swift
// 仅有一个闭包参数，使用尾随闭包写法
let r = arr.filter { $0 % 2 == 0 }

// 有两个闭包参数，则不使用尾随闭包写法
arr.forEach(where: { $0 % 2 == 0 }, body: { print($0) })
```

Bad

```swift
let r = arr.filter({ $0 % 2 == 0 })

arr.forEach(where: { $0 % 2 == 0 }) { print($0) }
```

​	

#### 闭包声明时，不需要写参数名，只声明类型即可

Good

```swift
func myFunc(completion: (Data) -> Void) {
}
```

Bad

```swift
func myFunc(completion: (_ data: Data) -> Void) {
}
```

​	

#### 使用[weak self]修饰的闭包，闭包开始判断self的有效性

```swift
fetchList(param) { [weak self] lst in
    guard let self = self else {
        return
    }
    // code
}
```

​	

#### 过滤、转换等，优先使用filter、map等高阶函数简化代码

Good

```swift
let arr = [1, 2, 3, 4]
let total = arr.reduce(0, +)
```

Bad

```swift
let arr = [1, 2, 3, 4]
var total = 0
for i in arr {
    total += i
}
```

​	

#### 优先使用let定义变量，而不是var

​	

#### 能推断出来的类型，不需要加类型限定

Good

```swift
let str = "Hello"
view.backgroundColor = .red
```

Bad

```swift
let str: String = "Hello"
view.backgroundColor = UIColor.red
```

​	

#### 变量声明时，使用简化写法。

Good

```swift
var m = [Int]()
```

Bad

```swift
var n = Array<Int>()
```

​	

#### 单行注释，优先使用 //

Good

```swift
print("Hello")	// Hello
```

Bad

```swift
print("Hello")	/* Hello */
```

​	

#### 异常的分支，提前用guard结束。

Good

```swift
func process(value: Int) {
    guard value > 0 else {
        return
    }

    // code
}
```

Bad

```swift
func process(value: Int) {
    if value > 0 {
        // code
    }
}
```

​	

#### 多个嵌套条件，能合并的，就合并到一个if中

Good

```swift
func process(v1: Int, v2: Int) {
    if v1 > 0, v2 > 0 {
        // code
    }
}

```

Bad

```swift
func process(v1: Int, v2: Int) {
    if v1 > 0 {
        if v2 > 0 {
            // code
        }
    }
}
```

​	

#### 尽可能使用private、fileprivate来限制作用域

Good

```swift
class MyClass {
    private func util() {   // 仅在类内部使用
    }
}
```

Bad

```swift
class MyClass {
    func util() {   // 仅在类内部使用
    }
}
```

​	

#### 尽量省略self，必要时才加

Good

```swift
extension Array where Element == Int {
    func myFunc() -> Int {
        return filter { $0 > 10 }.count
    }
}
```

Bad

```swift
extension Array where Element == Int {
    func myFunc() -> Int {
        return self.filter { $0 > 10 }.count
    }
}
```

​	

#### 不使用强制解包

Good

```swift
if let value = optional {
    // code
}
```

Bad

```swift
let value = optional!
```

​	

#### 不使用强制类型转换

Good

```swift
if let r = value as? String {
    // code
}
```

Bad

```swift
let r = value as! String
```

​	

#### 不使用try!

Good

```swift
let r = try? decodeData()
```

Bad

```swift
let r = try! decodeData()
```

​	

#### 不使用隐式解包

Good

```swift
let opt: String?
```

Bad

```swift
let opt: String!
```

