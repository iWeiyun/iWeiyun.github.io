---
title: Swift函数式编程探索
date: 2018-11-19 20:16:54
author: sniper
tags: swift,functional
---



以前在iOS上，除了RAC，比较少有函数式编程方面的实践。swift对函数式做了较多的支持，随着swift的普及，在iOS社区，函数式编程被越来越多的开发者所接受。并且因为函数式编程的一些优点，也越来越多的语言开始支持函数式的开发范式。

因为最近也在项目中开始实践函数式编程，也能逐渐感受到函数式强大之处。目前也有一点心得，本文就谈下自己对函数式编程的理解。
​	
不过在开始之前，先推荐个小段子：[好久不见](http://m.youku.com/video/id_XMzUwMzg4NDAyNA==.html?spm=a2h0k.8191407.0.0&source=&sharetype=2&from=groupmessage&isappinstalled=0)

​		

------



### 编程范式

在讲函数式编程之前，我们先来看看这两大编程范式：

- 命令式
- 声明式

来粘贴一个关于它们的定义：

> 命令式编程通过一系列改变程序状态的指令来完成计算。命令式编程模拟电脑运算，是行动导向的，关键在于定义解法，即“怎么做”，因而算法是显性而目标是隐性的。
>
> 声明式编程只描述程序应该完成的任务。声明式编程模拟人脑思维，是目标驱动的，关键在于描述问题，即“做什么”，因而目标是显性而算法是隐性的。

这两种范式的关键区别就在于，一个主要表达“怎么做”，一个主要表达“做什么”。通过两段代码，可以直观的感受一下它们的区别：
![命令式](/images/swift-functional/1542568661_33.png)
![声明式](/images/swift-functional/1542568666_91.png)
这两段代码都是做的同一件事，把数组里每个元素都乘2，再打印出来。
可以看出来，我们平时使用的主要都是命令式的写法，包括使用的最多的面向对象编程(OOP)，其实还是属于命令式编程的范畴。而这篇文章要讲的函数式编程，则是属于声明式编程。

​	

------



### 函数式编程的优势

- 减轻程序员思考的负担（防秃），降低出错的可能性

就拿上面举的例子来说，先不说一眼望去，命令式写法的复杂性就明显要高得多，即使是一个如此简单的逻辑，也容易写出错误。
当时写这个例子的时候，为了突出它们的区别，我故意没用for in range这种比较常规的写法，而使用了while，自己维护index。然鹅第一次跑起来就死循环了，再检查，发现是红框里的index加1漏了写。
函数式写法中，因为没有了状态的维护，杜绝了这种错误的发生。

- 代码可读性高
- 更简洁、更少的代码

看回定义，声明式编程描述“做什么”，目标是显性而算法是隐性的。而阅读代码时我们往往只关心代码做了什么，而不关心怎么做，这与声明式编程的思想是不谋而合的，所以函数式编程具有更高的代码可读性。
同时为了实现这种思想，算法的实现往往是内建的，或是隐藏的，这也让函数式的实现具有更少的代码，看起来更简洁。

- 适用于并发环境
- 易于优化

函数式编程中所使用的函数，大多要求为纯函数，这里可以先理解为数学意义上的函数，在相同的参数输入的情况下，一定会有同样的输出。
这一特点决定了纯函数可以并行的调用，而无需任何修改，因为无论并行与否，它都能输出正确的结果。
![并发](/images/swift-functional/1542597555_48.png)
上图parallelMap函数的实现就是将数组分成多段，然后在多个线程中分别调用map，最后再拼回一整个。通过这一点改动，就简单的对这个程序进行了多线程的性能优化。

- 细粒度的重用（函数级别）
- 易于测试

纯函数除了之前说的，还要求“没有任何可观察的副作用，不依赖外部环境的状态”。
后半还是容易理解的，因为只要依赖了外部环境，就不能保证相同输入得到相同输出了。
前半所说的副作用，是指除了返回值外，函数还通过其他方式对调用环境产生了影响，例如修改全局变量，写文件，print到控制台等等。
纯函数很容易进行单元测试，只需关心输入输出即可。重用也是同理，不必考虑调用的顺序，不用担心影响后续逻辑，放心的复用。
![函数重用](/images/swift-functional/1542614984_40.png)

- 易于重构

原因也就是前面说的那些，纯函数之间的依赖关系就是一个树，很容易理清。而对象之间的依赖就复杂得多，做过重构的同学应该都有体会，如果没有把代码看的很熟，是不敢轻易去动的。书里有一句话总结的很到位：
![FP和OO](/images/swift-functional/1542615515_42.png)

​	

-----



### 函数式编程指南

概念都了解了，那要怎么应用到实际的项目中呢？只要在编写代码时，往下面这几点靠，自然就能写出函数式风格的代码了：

- 只定义纯函数
- 不要用可变量
- 递归代替迭代
- 尽量少的数据结构
- 闭包、高阶函数的应用
- 尽可能使用内置的数据结构
- 尽可能使用内置的函数式工具

但是并非所有功能都适合用函数式来编写，很多情况下副作用是无法避免的：

> 函数式语言和逻辑式语言擅长基于数理逻辑的应用，命令式语言擅长基于业务逻辑的、尤其是交互式或事件驱动型的应用。

现阶段遇到这些情况时，还是不要强行应用函数式了，特别是苹果的系统框架还是基于面向对象的。

​	
接下来举个🌰：
我们项目中有这样一个需求，在一个列表中，有多种元素（文件、文件夹、笔记等）。还有多种操作，每种操作支持的类型、数量不同。现在选中了一些元素，求能对这些元素进行哪些操作。
例如，选中一个文件、一个文件夹，这时可以进行移动操作（批量、不支持笔记），但不能进行重命名操作（只支持单个）。
这是一个很常见的需求，并且不涉及UI、交互，适合用函数式来改造一下。

​	

最终我们需要这样一个东西，输入选择的元素列表，得到支持的操作集合。我们把它定义成Strategy类型，是一类函数，在函数式编程里，函数就是一等公民。

```swift
typealias Strategy = (_ items: [Any]) -> Set<WYOperationType>
```

光有Strategy还不够，我们要直接实现这样一个函数还是太过复杂，而且也不够灵活，需要把它拆解为更小的单元。
我采用的方法是，先定义一个默认的操作集合，包含了最常规的操作。然后定义一些Modifier来修改默认的操作集合。不同类型的元素支持哪些操作就定义在不同的Modifier里。

```swift
static let defaultStrategy: Strategy = { _ in
	[.delete, .share, .safeBoxMoveIn,
     .safeBoxMoveOut, .shareDirWithFriend, .rename,
     .download, .jumpToDir, .openIn,
     .favour, .move, .inbox, .unInbox,
     .fileInfo, .ocr, .groupMove,
     .noteGroup, .genPDF, .genGIF]
}

typealias Modifier = (_ items: [Any], _ operations: Set<WYOperationType>) -> Set<WYOperationType>

class func modify(_ strategy: @escaping Strategy, _ modifier: @escaping Modifier) -> Strategy {
    return { items in
        modifier(items, strategy(items))
    }
}
```

这里的modify函数，其实就是一个带items参数的compose（组合）操作。



下面我们就来定义一个目录的modifier：

```swift
static let dirSupported: Operand = { items in
    let dirs = items.lazy.compactMap { $0 as? WeiyunDir }
    return dirs.first.map {
        if $0.isCollecting() {
            return [.share, .shareDirWithFriend, .safeBoxMoveIn, .move, .inbox, .unInbox, .delete, .rename]
        } else {
            return [.share, .shareDirWithFriend, .safeBoxMoveIn, .move, .inbox, .delete, .rename]
        }
    }
}

static let dirModifier: Modifier = buildModifier(dirSupported, intersection)
```

dirSupported判断items中是否有目录，有的话就返回目录支持的操作，通过buildModifier和intersection，构造了一个“如果含有目录，就将目前的操作集合与目录支持的操作集合取交集”的一个Modifier。
考虑到Modifier其实也就只有支持、不支持、添加三种情况，分别对应集合的交、差、并集，就通过“返回操作集合”的Operand函数，和三种SetOperation来build一个Modifier，将Modifier的逻辑也细分：

```swift
typealias SetOperation = (_ first: Set<WYOperationType>, _ second: Set<WYOperationType>) -> Set<WYOperationType>

static let union: SetOperation = { $0.union($1) }
static let subtracting: SetOperation = { $0.subtracting($1) }
static let intersection: SetOperation = { $0.intersection($1) }

typealias Operand = (_ items: [Any]) -> Set<WYOperationType>?

class func buildModifier(_ operand: @escaping Operand, _ setOperation: @escaping SetOperation) -> Modifier {
    return { items, operations in
        if let ret = operand(items) {
            return setOperation(operations, ret)
        }
        return operations
    }
}
```

下面定义了几个用于将Modifier进行组合的函数：

```swift
static let emptyModifier: Modifier = { $1 }

class func concat(_ modifierLeft: @escaping Modifier, _ modifierRight: @escaping Modifier) -> Modifier {
    return { items, operations in
        return modifierRight(items, modifierLeft(items, operations))
    }
}

class func concatModifiers(_ modifiers: [Modifier]) -> Modifier {
    return modifiers.reduce(emptyModifier) { concat($0, $1) }
}
```



有了Modifier的compose函数，就可以定义更复杂的文件和笔记的Modifier了。
最后还有一个count的Modifier，是专门用来在多选的情况下，去除只支持单元素的操作。

```swift
static let fileSupported: Operand = { items in
    let files = items.lazy.filter { !($0 is WeiyunNote) }.compactMap { $0 as? WeiyunFile }
    return files.first.map { _ in
        [.download, .share, .shareDirWithFriend,
         .fileInfo, .safeBoxMoveIn, .move,
         .jumpToDir, .delete, .favour,
         .rename, .openIn, .ocr,
         .groupMove, .genGIF]
    }
}

static let fileUnsupported: Operand = { items in
    let files = items.lazy.filter { !($0 is WeiyunNote) }.compactMap { $0 as? WeiyunFile }
    let noOcr = files.filter { !$0.isShouldShowOCRAction() }.first
    let noImgVid = files.filter { !$0.isImageFile() && !$0.isVideoFile() }.first
    let noLivePhoto = files.filter { !$0.isLivePhoto }.first
    let secondFile = files.dropFirst().first

    let result: [WYOperationType] = (noOcr != nil ? [.ocr] : []) + (noImgVid != nil ? [.groupMove, .genGIF] : []) + (noLivePhoto != nil ? [.genGIF] : []) + (secondFile != nil ? [.genGIF] : [])
    return Set<WYOperationType>(result)
}

static let fileModifier: Modifier = concatModifiers([buildModifier(fileSupported, intersection), buildModifier(fileUnsupported, subtracting)])

static let noteSupported: Operand = { items in
    let notes = items.lazy.compactMap { $0 as? WeiyunNote }
    return notes.first.map { _ in
        return [.share, .delete, .noteGroup, .favour]
    }
}

static let noteUnsupported: Operand = { items in
    let notes = items.lazy.compactMap { $0 as? WeiyunNote }
    return notes.dropFirst().first.map { _ in
        return [.favour]
    }
}

static let noteModifier: Modifier = concatModifiers([buildModifier(noteSupported, intersection), buildModifier(noteUnsupported, subtracting)])

static let countUnsupported: Operand = { items in
    let wyItems = items.lazy.compactMap { $0 as? WeiyunItem }
    return wyItems.dropFirst().first.map { _ in
        return [.ocr, .fileInfo, .rename, .jumpToDir, .openIn, .inbox, .unInbox]
    }
}

static let countModifier: Modifier = buildModifier(countUnsupported, subtracting)
```

最后定义一个applyModifiers来应用一组Modifier：

```swift
class func applyModifiers(_ strategy: @escaping Strategy, _ modifiers: [Modifier]) -> Strategy {
    return modifiers.reduce(strategy) { modify($0, $1) }
}

static let filesTabStrategy: Strategy = applyModifiers(defaultStrategy, [dirModifier, fileModifier, noteModifier, countModifier])
```

filesTabStrategy就可以拿到我们APP的文件列表去用了，选中一些项目后，根据Strategy返回的操作集合来展示操作菜单。
如果其他场景有不同的需求，可以编写自己的Modifier，再与现有的一顿组合即可。



主要思想就是把函数、闭包作为数据来操作，运用于高阶函数。还要熟悉函数式编程的三板斧（map、filter、reduce）和lazy等函数式工具，遵循开头的那几个要点来编写代码，就算是入了函数式的大门了。

​	

-----



### 其他

- 纯函数的引用透明
  引用透明网上有很多种解释，函数式里，是指函数运行的结果与函数本身，是可以互相替代的。显然纯函数是具有引用透明性的，所以可以延迟执行而不影响结果，这是lazy能正确执行的理论前提。

  这里给出一个比较有意思的lazy filter的实现：
  ![LazyArray](/images/swift-functional/1542626381_77.png)

- 记忆
  可以简单的对纯函数进行缓存，提高性能。
  Swift里虽然没有内建的实现，但是自己实现一个也很简单：
  ![memorize](/images/swift-functional/1542626458_75.png)
  生产环境使用时还要考虑内存占用、淘汰等。

- 递归代替迭代
  ![递归](/images/swift-functional/1542626659_3.png)
  这里实现的是“每隔1、2、3、4……个数，取一个数”。
  不能使用var时，我们需要将迭代换成递归实现。好处就不再赘述了，并且符合函数式的原则。
  这里需要注意的是要用上尾递归，防止爆栈。
  然而我在swift上测试时，还是出现了爆栈的情况，网上查了资料说是swift不保证尾递归优化，好吧……

- let的性能问题
  ![插入百万数据](/images/swift-functional/1542627283_39.png)
  ![将十个数据插入百万元素的字典](/images/swift-functional/1542627287_75.png)
  纯函数式语言大概是不会出现这样的情况的，运行时的优化，数据结构的高度优化，结果应该是跟用var差不多的性能才对。至少以我的知识，都能实现一个插入时间复杂度为O(log n)的不可变字典。
  swift估计是没有做这方面的优化，赋值给var时应该是发生了拷贝。并且let的dict也没有插入一个值，返回一个新let dict的方法。

- 运算符重载
  ![运算符重载](/images/swift-functional/1542627988_47.png)

- ReactiveX

将Async、Lazy、Multi-threading版本的函数式工具，引入了面向对象编程，非常值得使用。

​	

-----



### 总结

虽然说swift引入了许多函数式编程的东西，光从“没有尾递归优化”和“不可变数据结构优化”这两点来看，实际上还是不能完全按照纯函数式语言的那一套来编码的。
函数式编程，现阶段也还不会在项目中大量运用，但是这种思维，确实可以给我们平时的编码带来不一样的启发。
以目前swift的能力来说，一些比较简单的函数式的应用还是可以胜任的。
即使在OO的编程中，这些函数式的工具也能够对简化代码、逻辑起到很大的作用。

​	

----



### 参考资料
[《函数式编程思维》](https://www.zhihu.com/pub/reader/119565028)
[《Swift函数式编程》](https://objccn.io/products/functional-swift/)