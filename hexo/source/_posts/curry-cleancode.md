---
title: 利用柯里化去除重复代码
date: 2018-09-04 19:58:01
author: matthew
Tags: swift,curry,柯里化,重构,重复
---



**Swift中，函数是一等公民**

#### 问题

最近因为某个类中有重复代码，在“固化思维”重构之后，虽然原来的重复代码去掉了，但又有如下样式的代码，仔细想想，其实还是有重复，如files和dirs的获取，以及对结果的处理，代码是完全一样的。

<!-- more -->

```swift
extension Array where Element: WeiyunItem {
    fileprivate func restore(dir: WeiyunDir?) -> Completable {
        let files = compactMap { $0 as? WeiyunFile }
        let dirs = compactMap { $0 as? WeiyunDir }

        return Completable.create { observer -> Disposable in
            WeiyunSDK.sharedInstance()?.restoreRecycleFile(files, dir: dirs, pdirkey: dir?.dirkey, ppdirkey: dir?.pdirkey, block: { _, _, err in
                if case let error as NSError = err {
                    observer(.error(error))
                } else {
                    observer(.completed)
                }
            })
            return Disposables.create()
        }
    }

    fileprivate func delete() -> Completable {
        let files = compactMap { $0 as? WeiyunFile }
        let dirs = compactMap { $0 as? WeiyunDir }

        return Completable.create { observer -> Disposable in
            WeiyunSDK.sharedInstance()?.clearRecycleFile(files, dir: dirs, block: { _, _, err in
                if case let error as NSError = err, !ignoreError(error) {
                    observer(.error(error))
                } else {
                    observer(.completed)
                }
            })
            return Disposables.create()
        }
    }
}
```



#### 解决

按传统的思路来写的话，就是将相同的代码抽取到函数里，然后再调用相应函数来避免重复代码。 重构后大概的代码如下：

```swift
extension Array where Element: WeiyunItem {
    private func splitItems() -> ([WeiyunFile], [WeiyunDir]) {
        let files = compactMap { $0 as? WeiyunFile }
        let dirs = compactMap { $0 as? WeiyunDir }
        return (files, dirs)
    }

    private func processResult(observer: PrimitiveSequenceType.CompletableObserver, err: Error?) {
        if case let error as NSError = err {
            observer(.error(error))
        } else {
            observer(.completed)
        }
    }

    fileprivate func restore(dir: WeiyunDir?) -> Completable {
        let tuple = splitItems()
        return Completable.create { observer -> Disposable in
            WeiyunSDK.sharedInstance()?.restoreRecycleFile(tuple.0, dir: tuple.1, pdirkey: dir?.dirkey, ppdirkey: dir?.pdirkey, block: { _, _, err in
                self.processResult(observer: observer, err: err)
            })
            return Disposables.create()
        }
    }

    fileprivate func delete() -> Completable {
        let tuple = splitItems()
        return Completable.create { observer -> Disposable in
            WeiyunSDK.sharedInstance()?.clearRecycleFile(tuple.0, dir: tuple.1, block: { _, _, err in
                self.processResult(observer: observer, err: err)
            })
            return Disposables.create()
        }
    }
}
```

看起来还ok，除了Completable.create及Disposables.create()之外，基本没有重复代码了。



#### 继续思考

不过，这是终点了吗？并不是，毕竟还有部分代码是重复的。

我们现在换一种思路来思考，第一张截图里面，除了调用的WeiyunSDK的接口不同，传入的参数不同，其它所有代码都是一样的，那么是否可以在这儿做文章？

再回到开头看下这句话：Swift中函数是一等公民。这句话的意义是说函数也可以被操作、变换、处理等，你想到的基本都能做。

那么，把函数作为值传入处理函数中，在处理函数中调用处理就ok。通过传入不同的函数，即可实现调用不同的请求。

但有个很大的问题，函数类型不一样，restoreRecycleFile多了第2、3两个参数！

如何把restoreRecycleFile和clearRecycleFile变为具有相同参数的函数，就是要解决的问题。

今天的主角：**柯里化**，就是来解决这个问题的。



#### 柯里化

柯里化是一个通用的概念，在函数式编程里面非常重要。它在维基上的定义是：

> 把接受多个参数的函数变换成接受一个单一参数（最初函数的第一个参数）的函数，并且返回接受余下的参数而且返回结果的新函数的技术。



就作用来说，柯里化可以改变函数类型，可以提前绑定其中的参数。



Github上也有一些现成的柯里化开源库，可以直接用的。如[Curry](https://github.com/thoughtbot/Curry)、[Prelude](https://github.com/robrix/Prelude)等



可以看如何将两个参数变一个参数的简单实现，以及如何使用柯里化：

```swift
public func curry<A, B, C>(_ function: @escaping ((A, B)) -> C) -> (A) -> (B) -> C {
    return { (a: A) -> (B) -> C in { (b: B) -> C in function((a, b)) } }
}

// 自定义函数
func myAdd(a: Int, b: Int) {
    return a + b
}

let f = curry(myAdd)(5)	// 这样就可以变为只接受一个参数的函数，
f(10)	// 可以这样来调用，并且结果是15
```



#### 最终方案

在我们这个需求场景中，是需要提前绑定第2和3个参数，并且返回只接受三个参数的函数，这些开源库没有提供相应实现，不自己实现一个并不复杂：

```swift
// 可以绑定2、3参数的curry化函数
private func curry2_3<A, B, C, D, E, F>(_ function: @escaping (A, B, C, D, E) -> F) -> (C, D) -> (A, B, E) -> F {
    return { c, d in { a, b, e in function(a, b, c, d, e) } }
}
```



然后把这个函数用上之后，就可以将代码整理成如下：

```swift
fileprivate extension Array where Element: WeiyunItem {
    func restore(dir: WeiyunDir?) -> Completable {
        let f = curry2_3(WeiyunSDK.sharedInstance().restoreRecycleFile)(dir?.dirkey, dir?.pdirkey)
        return operate(f)
    }

    func delete() -> Completable {
        let f = WeiyunSDK.sharedInstance().clearRecycleFile
        return operate(f)
    }

    private func operate(_ function: @escaping ([WeiyunFile]?, [WeiyunDir]?, RestoreRecycleItemBlock?) -> Void) -> Completable {
        let files = compactMap { $0 as? WeiyunFile }
        let dirs = compactMap { $0 as? WeiyunDir }

        return Completable.create { observer -> Disposable in
            function(files, dirs, { _, _, err in
                if case let error as NSError = err {
                    observer(.error(error))
                } else {
                    observer(.completed)
                }
            })

            return Disposables.create()
        }
    }
}
```

这样就没有任何重复代码了



这儿只是演示了柯里化非常简单的一种使用场景，在函数式编程中，对函数的处理变换无处不在，柯里化也会大放异彩！

