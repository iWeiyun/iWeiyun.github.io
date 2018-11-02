---
title: RxCocoa简单源码分析
author: matthew
date: 2018-11-01 21:38:28
tags: rxswift,rxcocoa,bind,ios,swift
---



### 前言

RxCocoa是[RxSwift](https://github.com/ReactiveX/RxSwift)的一部分，主要是UI相关的Rx封装。比如实现了很多组件的绑定功能，简化处理逻辑。也可以监听delegate改变，无须把控件创建及delegate处理分开写等。

RxCocoa里面也定义了很多类，专门为UI处理提供的，比如[ControlProperty](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/observable_and_observer/control_property.html)、[ControlEvent](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/observable/control_event.html)、[Driver](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/observable/driver.html)、[Binder](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/observer/binder.html)等。RxCocoa可以用好的话，可以极大简化UI相关处理逻辑。但是，要想随心所欲的使用，还是要对其实现要有一定的了解，否则就容易写出不是那么简洁的代码。

比如前几天在一篇博客上，看到的一段代码：

```swift
textfield.rx.text
    .asObservable()
    .subscribe { print($0) }
    .disposed(by: disposeBag)
```

这段代码一眼看过去，是没什么问题的，执行起来看起来也是ok的。

但是，这段代码的问题就是存在冗余。具体为什么冗余，稍后再分析。先从简单的示例了解RxCocoa

​	



### 简单示例

[RxSwift](https://github.com/ReactiveX/RxSwift)的源码里面有附带示例代码，源码clone下来之后，打开Rx.xcworkspace，即可以选择示例运行看效果。

现在就拿最简单的Numbers例子看下，(也可以在线看下这个最简单示例代码：[Numbers](https://github.com/ReactiveX/RxSwift/blob/master/RxExample/RxExample/Examples/Numbers/NumbersViewController.swift)）



核心代码就这一段：

```swift
Observable.combineLatest(number1.rx.text.orEmpty, number2.rx.text.orEmpty, number3.rx.text.orEmpty) { textValue1, textValue2, textValue3 -> Int in
		return (Int(textValue1) ?? 0) + (Int(textValue2) ?? 0) + (Int(textValue3) ?? 0)
	}
    .map { $0.description }
    .bind(to: result.rx.text)
    .disposed(by: disposeBag)
```

这个示例是将三个输入框的内容加起来，绑定在Label上，上面值变化之后，下面的Label立即跟着变化。



为了让例子更简单，我们可以只把label的值绑定在第一个输入框上：

```swift
number1.rx.text.orEmpty
    .map { $0.description }
    .bind(to: result.rx.text)
    .disposed(by: disposeBag)
```



现在我们逐个分析下，这里面的 rx、text、orEmpty、bind到底是什么

​	



### rx

number1.rx是表示什么呢？我们根据源码来推导一下

先看下rx属性的源码实现：

```swift
public var rx: Reactive<Self> {
    get {
        return Reactive(self)
    }
    set {
        // this enables using Reactive to "mutate" base object
    }
}
```

实际上是创建了一个Reactive对象，所以：

```
number1.rx  ===>  Reactive(number1)
```



Reactive源码又如下：

```swift
public struct Reactive<Base> {
    /// Base object to extend.
    public let base: Base

    /// Creates extensions with base object.
    ///
    /// - parameter base: Base object.
    public init(_ base: Base) {
        self.base = base
    }
}
```



所以number1.rx最终会变成如下代码：

```swift
public struct Reactive {
    public let base: UITextField	// 指向number1
}
```



所以number1.rx是变成了**Reactive结构体**，此时，Reactive的扩展方法，我们就可以使用了。

所以，这个.rx是进入Rx世界的入口，控件调用.rx属性之后，后面的内容就表示进入了Rx的世界了。

​	



### text

上面分析了，number1.rx是一个Reactive的结构体，它后面就可以继续调用Reactive及其扩展的属性和方法了。

（实际上number1.rx只能调用不限定的扩展的方法或者限定Base是UITextField类型的扩展方法）

所以，text属性就是Reactive的扩展的属性。这个属性定义在UITextField+Rx.swift文件中，下面是其简化后的实现：

```swift
extension Reactive where Base: UITextField {
    public var text: ControlProperty<String?> {
        return value
    }

    public var value: ControlProperty<String?> {
        // 默认事件支持了allEditingEvents、valueChanged
        return base.rx.controlPropertyWithDefaultEvents(
            getter: { textField in			// 要发出事件时，所需的数据，即是从这儿获取
                textField.text
        	},
            setter: { textField, value in	// 作为监听者时，收到数据时会调到这儿
                if textField.text != value {
                    textField.text = value
                }
        	}
        )
    }
}
```



text简单的调用了value属性，它们是ControlProperty类型的。

ControlProperty是表示控件属性，即能监听变化，又能发出通知，即同时实现了ObservableType和ObserverType协议，所以控件才能支持**双向绑定**。

上面代码中的getter块是在作为ObservableType时所使用的，setter块是作为ObserverType所需要的。

具体ControlProperty实现就不展开了，可以在UIControl+Rx.swift查看具体实现。

​	



### orEmpty

上面代码中的一个细节：

```swift
public var text: ControlProperty<String?>
```

这个value值传出来之后是String?的，是个可选值。在很多情况下，并不想要可选值，只希望如果为nil时，传""空字符串出来即可。否则可选值出来之后，外面可能处理为nil的情况处理起来会比较繁琐。

 orEmpty就是这个作用。



orEmpty的实现如下：

```swift
public var orEmpty: ControlProperty<String> {
        let original: ControlProperty<String?> = self.asControlProperty()

        let values: Observable<String> = original._values.map { $0 ?? "" }
        let valueSink: AnyObserver<String> = original._valueSink.mapObserver { $0 }
        return ControlProperty<String>(values: values, valueSink: valueSink)
    }
```

会根据目前的values构造一个新的values，并最终构造一个新的ControlProperty，去除可选值的情况。

​	



### bind

bind即是用来将一个信号发送者和一个信号监听者绑定在一起，即有信号发送，监听者自动收到通知。听起来是跟subscribe做的事情比较类似。

bind(to:)有几个不同的实现，最简单的实现版本如下：

```swift
public func bind<O: ObserverType>(to observer: O) -> Disposable where O.E == E {
    return self.subscribe(observer)
}
```

其实就是订阅的封装。可以理解为为subscribe起了一个在绑定场景下比较容易理解的名字。

在很多情况下，我们的observer并不需要处理error、complete事件，并且处理逻辑需要在主线程中执行。所以RxCocoa帮我们封装好了一个叫Binder的对象，我们使用这个对象时，不需要考虑太多。

RxCocoa提供的一些监听属性，比如UILabel的rx.text属性即是Binder类型的:

```swift
extension Reactive where Base: UILabel {    
    public var text: Binder<String?> {
        return Binder(self.base) { label, text in
            label.text = text	// 收到事件就会调用到这儿
        }
    }
}
```

​	



### 结尾

这样这个简单示例就串起来了，并且能够很明确每次调用在做什么事情。

现在再看最开始的那段代码：

```swift
textfield.rx.text
    .asObservable()
    .subscribe { print($0) }
    .disposed(by: disposeBag)
```

我们知道textfield.rx.text是ControlProperty的类型，并且这个类型本身就实现了Observable协议的，那么冗余的代码就很明确了，即asObservable()这一次调用是没意义的。

