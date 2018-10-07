---
title: 利用函数组合提升代码可维护性
date: 2018-10-03 22:24:56
author: matthew
tags: swift,函数式,函数组合,rxswift,promise
---



### 前言

函数组合，在函数式编程里面也是挺重要的概念，能够将函数进行操作合并等，在有些场景下可以大幅度提升代码的可读及可维护性。

下面就演示一些利用函数组合重构代码以达到更好可维护性的例子



### 简单场景

假设有如下代码： 

```swift
process1(_ param: String) -> String
process2(_ param: String) -> String
process3(_ param: String) -> String
process4(_ param: String) -> String
```

这些函数来处理字符串，如果要组合调用的话，可能会写出来如下的代码：

```swift
var str = ... 
str = process1(str) 
str = process2(str) 
str = process3(str) 
str = process4(str) 
// use str
```

或者更洒脱一些，写出如下的代码：

```swift
let ret = process4(process3(process2(process1(str))))
```



第二种方式可读性不算太好，第一种方式代码写起来又会非常繁琐。那应该如何来优化呢？



### 优化

Swift中是支持**自定义运算符**的，而且swift中**函数是一等公民**。这两个特性是很强大的，利用他们，可以更好的实现函数的组合，可以让代码看起来更简洁、更易读。

大概的思路是把process1、process2等进行组合，组合成一个新的函数，调用这个新函数的效果，跟分开挨个调用是一样的。

优化后的代码如下：

```swift
infix operator ++ : AdditionPrecedence
func ++ (lhs: @escaping (String) -> String, rhs: @escaping (String) -> String) -> (String) -> String {
    return { rhs(lhs($0)) }
}

let ret = (process1 ++ process2 ++ process3 ++ process4)(str)
```

这样写出来的代码，易读且易维护，要增删操作、调整调用顺序等都是很容易的。



### 更多场景

上面这种场景，是比较特殊的场景，函数签名一致并且是同步函数。在真正工作中更普遍的场景是：

1. 函数签名不一致，如process1(String)，process2(Int, String)
2. 函数是异步操作，而且回调的闭包类型也不一样等。



#### 函数签名不一致

要能组合函数类型不一致的问题，可以参考：[利用柯里化去除重复代码](https://iweiyun.github.io/2018/09/04/curry-cleancode/)，利用柯里化 (严格来说叫partial function application) 可以很容易解决。

代码示例如下：

```swift
process1(_ param: String) -> String
process2(_ i: Int, _ param: String) -> String
process3(_ i: Int, _ param: String) -> String
process4(_ i: Int, _ param: String) -> String

process1 ++ curry(process2) ++ curry(process3) ++ curry(process4)
```

不过这儿补充下，有柯里化，就有**反柯里化**。反柯里化就是给函数增加参数，让该函数跟其它函数类型对齐。

反柯里化的一种简单实现如下：

```swift
func uncurry(function: @escaping (String) -> String) -> (String, Int) -> String {
    return { s, _ in function(s) }
}
```

利用该反柯里化方式，新的组合代码可以适度简化为这样：

```swift
uncuryy(process1) ++ process2 ++ process3 ++ process4
```

uncurry完善的实现，可以参考Github上的一些实现，如 [swift-overture](https://github.com/pointfreeco/swift-overture/blob/master/Sources/Overture/Uncurry.swift)



#### 异步操作

再来看异步操作的问题。

说到异步处理，如果熟悉一些异步处理框架，如[PromiseKit](https://github.com/mxcl/PromiseKit)或[RxSwift](https://github.com/ReactiveX/RxSwift)，那么可能知道PromiseKit里的Promise或RxSwift里的Observable这两个对象。

仔细想想，Promise和Observable本身就是很有意思的对象，这些对象可以封装异步操作，当然，也可以表示同步操作，表示纯数据等。这些对象本身也提供了很多操作，操作之后，返回的结果仍然是该对象类型。（在函数式编程里面，这两个对象都可以理解为[Monad](http://www.ruanyifeng.com/blog/2015/07/monad.html)对象）

理解上面这一点是关键，如果Observable本身可以封装异步操作，那么，一个异步操作就可以表达为一个同步函数，只是返回对象是一个代表同步或异步的对象。这样异步的问题就转变为同步处理的问题了。



下面继续举个简单的例子

假设有如下4个异步操作：

> asyncProcess1
> asyncProcess2_1
> asyncProcess2_2
> asyncProcess3

1、2、3这几个是并发，2_1和2_2是串行

用RxSwift写的传统代码大概如下：

```swift
asyncProcess1(param: [String]) -> Observable<Ret> {}
asyncProcess2_1(param1: Int, param2: [String]) -> Observable<Ret> {}
asyncProcess2_2(param: [String]) -> Observable<Ret> {}
asyncProcess3(param: [String]) -> Observable<Ret> {}

let process2 = Observable.concat(asyncProcess2_1(value, strs), asyncProcess2_2(strs))
let process = Observable.merge([asyncProcess1(strs), process2, asyncProcess3(strs)])
// some code

```



下面我们就尝试重构下该代码。

先定义下通用的concat和merge的操作符：

```swift
// 并发两个函数，合并成一个函数
infix operator ||| : RxPrecedence
// 串行两个函数，合并成一个函数
infix operator >>> : RxPrecedence

typealias RxOper<T, U> = (T) -> Observable<U>

func ||| <T, U>(lfun: @escaping RxOper<T, U>, rfun: @escaping RxOper<T, U>) -> RxOper<T, U> {
    return { value in
        Observable.merge([lfun(value), rfun(value)])
    }
}

func >>> <T, U>(lfun: @escaping RxOper<T, U>, rfun: @escaping RxOper<T, U>) -> RxOper<T, U> {
    return { value in
        lfun(value).concat(rfun(value))
    }
}
```



然后写相应的业务代码：

```swift
// 新的处理代码
let process = asyncProcess1 ||| (curry(asyncProcess2_1)(value) >>> asyncProcess2_2) ||| asyncProcess3
// process(strs)...
```



新代码的优势一目了然。并且这些例子都是拿的非常简单的示例来讲解的，真正的使用场景上，当操作数量逐渐增加，操作逻辑逐渐复杂时，传统的代码写法的冗余就越能显现。



### 参考资料

[Function Composition in Swift](https://github.com/ijoshsmith/function-composition-in-swift)

[RxSwift中文文档](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/)