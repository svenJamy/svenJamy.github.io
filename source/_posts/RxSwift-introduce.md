---
title: RxSwift系列笔记-入门
date: 2016-09-24 10:10:36
tags: RxSwift
---

## Introduction
RX是通过观察序列组成的异步和基于事件的程序，Rx 是Reactive Extensions的一部分，同系列的还有RXJava,RXJS等 [RX系列](http://reactivex.io/intro.html)。RX扩展了[观察者模式](https://en.wikipedia.org/wiki/Observer_pattern),支持数据和事件序列，并增加了一些操作使得我们可以将这些序列组成在一起。

首先由一点要说明的是，RX是函数式的，也是响应式的，但并不是真正意义上的Functional Reactive Programming[FRP](https://github.com/conal/talk-2015-essence-and-origins-of-frp)。
<!-- more -->
### Why RXswift
在我们平时的开发过程中，当处理事件响应的时候，我们通常会通过不同的方式去处理，如使用KVO,Delegate监听属性值变化，`@IBAction`处理handle事件。这几种不同的方式处理无疑增加了代码的复杂度，RXSwift和RAC提供了这样的解决方案：把事件的产生和响应处理统一。

### Concepts
`Observable`是观察者模式中的被观察者对象，相当于一个事件`Sequence`序列，swift的`Sequence`可以异步的接收元素，这是RXSwift的本质，其他的都是基于这个扩展的。`observer`是观察者模式中的观察者对象(subscribers订阅者)。这两个是一一对应的，可观察者序列里面的闭包要有观察者订阅的时候才执行,但是如果有订阅者订阅了该信号，则会执行相应的闭包，订阅者就可以接收信号：

```

example("Observable with subscriber") {
  _ = Observable<String>.create { observerOfString in
            print("Observable created")
            observerOfString.on(.next("😉"))
            observerOfString.on(.completed)
            return Disposables.create()
        }
        .subscribe { event in
            print(event)
    }
}
--- Observable with subscriber example ---
Observable created
next(😉)
completed

```

事件信息主要分为下面3种：

* .Next(Element) 表示新的事件序列。
* .Completed 表示事件序列的结束。
* .Error 表示异常发生导致的序列结束。

## 创建和订阅`Observable`
### `never`
创建一个不会发送任何信号和结束的序列：

```
example("never") {
    let disposeBag = DisposeBag()
    let neverSequence = Observable<String>.never()
    
    let neverSequenceSubscription = neverSequence
        .subscribe { _ in
            print("This will never be printed")
    }
    
    neverSequenceSubscription.addDisposableTo(disposeBag)
}
> nothing
```

### `empty`
创建一个空的序列，只发送.completed信号：

```
example("empty") {
    let disposeBag = DisposeBag()
    
    Observable<Int>.empty()
        .subscribe { event in
            print(event)
        }
        .addDisposableTo(disposeBag)
}
> completed
```

### `just`
创建一个只包含一个元素的序列，它会先发送 .Next(Element) ，然后发送 .Completed：

```
example("just") {
    let disposeBag = DisposeBag()
    
    Observable.just("🔴")
        .subscribe { event in
            print(event)
        }
        .addDisposableTo(disposeBag)
}
--- just example ---
next(🔴)
completed
```

### `of`
创建一个包含固定元素的序列：

```
example("of") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐶", "🐱", "🐭", "🐹")
        .subscribe(onNext: { element in
            print(element)
        })
        .addDisposableTo(disposeBag)
}
--- of example ---
🐶
🐱
🐭
🐹
```
我们注意到订阅信号的时候没有使用之前的闭包，而是使用了`subscribe(onNext:)`便捷方法，之前的`subscribe(_:)`可以处理所有的事件类型(next,error,completed)，但是`subscribe(onNext:)`只会处理next事件，而忽略其他的事件。同理RXSwift还提供了`subscribe(onError:)`及`subscribe(onCompleted:)`快捷方法。
### `from`
把swift中的`Sequence`( `Array`, `Dictionary`,  `Set`)转化为事件序列，关于`Sequence`相关的介绍，请查看上一篇博客：

```
example("from") {
    let disposeBag = DisposeBag()
    
    Observable.from(["🐶", "🐱", "🐭", "🐹"])
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- from example ---
🐶
🐱
🐭
🐹
```
### `create`
可以通过create创建自定义的事件序列，在闭包里面通过.on方法添加事件：

```
example("create") {
    let disposeBag = DisposeBag()
    
    let myJust = { (element: String) -> Observable<String> in
        return Observable.create { observer in
            observer.on(.next(element))
            observer.on(.completed)
            return Disposables.create()
        }
    }
        
    myJust("🔴")
        .subscribe { print($0) }
        .addDisposableTo(disposeBag)
}
--- create example ---
next(🔴)
completed
```
### `range`
创建一个可以发送一系列连续整数的时间序列，最后发送.completed事件：

```
example("range") {
    let disposeBag = DisposeBag()
    
    Observable.range(start: 1, count: 10)
        .subscribe { print($0) }
        .addDisposableTo(disposeBag)
}
--- range example ---
next(1)
next(2)
next(3)
...
completed
```
### `repeatElement`
创建一个可以重复发送相同元素的事件序列：

```
example("repeatElement") {
    let disposeBag = DisposeBag()
    
    Observable.repeatElement("🔴")
        .take(3)
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- repeatElement example ---
🔴
🔴
🔴
```
take操作返回指定数量的事件序列开始的元素。
### `generate`
创建一个事件序列，发送到元素是符合指定的条件的：

```
example("generate") {
    let disposeBag = DisposeBag()
    
    Observable.generate(
            initialState: 0,
            condition: { $0 < 3 },
            iterate: { $0 + 1 }
        )
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- generate example ---
0
1
2
```
### `deferred`
为每一个订阅者创建一个新的事件序列，就是说每个订阅者订阅的对象都是内容相同而完全独立的序列：

```
    let disposeBag = DisposeBag()
    var count = 1
    
    let deferredSequence = Observable<String>.deferred {
        print("Creating \(count)")
        count += 1
        
        return Observable.create { observer in
            print("Emitting...")
            observer.onNext("🐶")
            observer.onNext("🐱")
            observer.onNext("🐵")
            return Disposables.create()
        }
    }
    
    deferredSequence
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
    
    deferredSequence
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- deferred example ---
Creating 1
Emitting...
🐶
🐱
🐵
Creating 2
Emitting...
🐶
🐱
🐵
```
### `error`
创建一个事件序列，不发送任何事件，而是直接返回.error然后结束。

## Subjects
subject可以看做是一种桥梁或者代理。在RXSwift中它既可以作为被观察者也可以是观察者。这就意味着subject可以订阅其他`Observable`对象，同时也可以向订阅者发送事件序列[更多](http://reactivex.io/documentation/subject.html)。在开始介绍案例之前先定义如下方法：

```
extension ObservableType {
    
    /**
     Add observer with `id` and print each emitted event.
     - parameter id: an identifier for the subscription.
     */
    func addObserver(_ id: String) -> Disposable {
        return subscribe { print("Subscription:", id, "Event:", $0) }
    }
    
}

func writeSequenceToConsole<O: ObservableType>(name: String, sequence: O) -> Disposable {
    return sequence.subscribe { event in
        print("Subscription: \(name), event: \(event)")
    }
}
```
### `PublishSubject`
向当前所有的订阅者发送新的事件，类似于广播~
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/publishsubject.png "PublishSubject")

```
example("PublishSubject") {
    let disposeBag = DisposeBag()
    let subject = PublishSubject<String>()
    
    subject.addObserver("1").addDisposableTo(disposeBag)
    subject.onNext("🐶")
    subject.onNext("🐱")
    
    subject.addObserver("2").addDisposableTo(disposeBag)
    subject.onNext("🅰️")
    subject.onNext("🅱️")
}
--- PublishSubject example ---
Subscription: 1 Event: next(🐶)
Subscription: 1 Event: next(🐱)
Subscription: 1 Event: next(🅰️)
Subscription: 2 Event: next(🅰️)
Subscription: 1 Event: next(🅱️)
Subscription: 2 Event: next(🅱️)
```
上面的事例中使用了`onNext(_:)`方法，等同于`on(.next(_:)`方法，向当前的订阅者发送一个next事件。类似的还有`onError(_:)` ,`onCompleted()`。
### `ReplaySubject`
和上面的`PublishSubject`类似，向所有的订阅者发送新的事件。不同的是`ReplaySubject`可以指定向新的订阅者发送之前的事件，参数`bufferSize `是缓冲区的大小，决定了补发事件的最大值：
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/replaysubject.png)

```
example("ReplaySubject") {
    let disposeBag = DisposeBag()
    let subject = ReplaySubject<String>.create(bufferSize: 1)
    
    subject.addObserver("1").addDisposableTo(disposeBag)
    subject.onNext("🐶")
    subject.onNext("🐱")
    
    subject.addObserver("2").addDisposableTo(disposeBag)
    subject.onNext("🅰️")
    subject.onNext("🅱️")
}
--- ReplaySubject example ---
Subscription: 1 Event: next(🐶)
Subscription: 1 Event: next(🐱)
Subscription: 2 Event: next(🐱)
Subscription: 1 Event: next(🅰️)
Subscription: 2 Event: next(🅰️)
Subscription: 1 Event: next(🅱️)
Subscription: 2 Event: next(🅱️)
```
### `BehaviorSubject`
也是向所有的订阅者发送新的事件，不同的是在新的订阅者订阅的时候会发送最近发送的事件，如果没有的话则发送默认值。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/behaviorsubject.png)

```
example("BehaviorSubject") {
    let disposeBag = DisposeBag()
    let subject = BehaviorSubject(value: "🔴")
    
    subject.addObserver("1").addDisposableTo(disposeBag)
    subject.onNext("🐶")
    subject.onNext("🐱")
    
    subject.addObserver("2").addDisposableTo(disposeBag)
    subject.onNext("🅰️")
    subject.onNext("🅱️")
    
    subject.addObserver("3").addDisposableTo(disposeBag)
    subject.onNext("🍐")
    subject.onNext("🍊")
}
--- BehaviorSubject example ---
Subscription: 1 Event: next(🔴)
Subscription: 1 Event: next(🐶)
Subscription: 1 Event: next(🐱)
Subscription: 2 Event: next(🐱)
Subscription: 1 Event: next(🅰️)
Subscription: 2 Event: next(🅰️)
Subscription: 1 Event: next(🅱️)
Subscription: 2 Event: next(🅱️)
Subscription: 3 Event: next(🅱️)
Subscription: 1 Event: next(🍐)
Subscription: 2 Event: next(🍐)
Subscription: 3 Event: next(🍐)
Subscription: 1 Event: next(🍊)
Subscription: 2 Event: next(🍊)
Subscription: 3 Event: next(🍊)
```
在上面的几个案例中，好像结果少了一个`completed `事件， `PublishSubject`, `ReplaySubject`, `BehaviorSubject` 默认不发送`completed `事件。
### `Variable`
`Variable`是基于`BehaviorSubject`的一层封装，所以也会向新的订阅者发送最近发送的事件，同样也维持当前值的状态。 `Variable`不会发送`Error`事件，但是会自动发送Completed事件并在`deinit`的时候释放。

```
example("Variable") {
    let disposeBag = DisposeBag()
    let variable = Variable("🔴")
  variable.asObservable().addObserver("1").addDisposableTo(disposeBag)
    variable.value = "🐶"
    variable.value = "🐱"   
  variable.asObservable().addObserver("2").addDisposableTo(disposeBag)
    variable.value = "🅰️"
    variable.value = "🅱️"
}
--- Variable example ---
Subscription: 1 Event: next(🔴)
Subscription: 1 Event: next(🐶)
Subscription: 1 Event: next(🐱)
Subscription: 2 Event: next(🐱)
Subscription: 1 Event: next(🅰️)
Subscription: 2 Event: next(🅰️)
Subscription: 1 Event: next(🅱️)
Subscription: 2 Event: next(🅱️)
Subscription: 1 Event: completed
Subscription: 2 Event: completed
```
在`Variable`实例中调用`asObservable()`方法可以访问`BehaviorSubject`事件序列，类似一个转换为可观察的作用。`Variable`没有实现on方法，但是提供了一个`value`方法，可以通过这个方法获取和设置当前的值，设置新值会添加到`BehaviorSubject `的事件队列中。

## Combination Operators
RXSwift提供了下面几种方法，可以把几种序列组成一个新的序列。
### `startWith`
在发送`Observable`源事件之前，发送指定的事件序列元素。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/startwith.png)

```
example("startWith") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐶", "🐱", "🐭", "🐹")
        .startWith("1️⃣")
        .startWith("2️⃣")
        .startWith("3️⃣", "🅰️", "🅱️")
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- startWith example ---
3️⃣
🅰️
🅱️
2️⃣
1️⃣
🐶
🐱
🐭
🐹
```
从上面的例子可以看出，`startWith`可以在后进先出的基础上链接。
### `merge`
把几个不同的`Observable`源序列中的元素组合成一个新的`Observable`源序列，新的`Observable`同样可以发送新的元素。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/merge.png)

```
example("merge") {
    let disposeBag = DisposeBag()
    
    let subject1 = PublishSubject<String>()
    let subject2 = PublishSubject<String>()
    
    Observable.of(subject1, subject2)
        .merge()
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
    
    subject1.onNext("🅰️")
    
    subject1.onNext("🅱️")
    
    subject2.onNext("①")
    
    subject2.onNext("②")
    
    subject1.onNext("🆎")
    
    subject2.onNext("③")
}
--- merge example ---
🅰️
🅱️
①
②
🆎
③
```
### `zip`
把几种不同的`Observable`序列源组合成一个新的`Observable`序列源，最多可以组合8个。并将从复合`Observable`序列发送每个源对应索引在`Observable`序列的元素，每个源序列在索引上都要有元素才发送。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/zip.png)

```
example("zip") {
    let disposeBag = DisposeBag()
    
    let stringSubject = PublishSubject<String>()
    let intSubject = PublishSubject<Int>()
    
    Observable.zip(stringSubject, intSubject) { stringElement, intElement in
        "\(stringElement) \(intElement)"
        }
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
    
    stringSubject.onNext("🅰️")
    stringSubject.onNext("🅱️")
    
    intSubject.onNext(1)
    
    intSubject.onNext(2)
    
    stringSubject.onNext("🆎")
    intSubject.onNext(3)
}
--- zip example ---
🅰️ 1
🅱️ 2
🆎 3
```
### `combineLatest`
和`zip`方法类似，把几种不同的`Observable`序列源组合成一个新的`Observable`序列源，最多可以组合8个。不同的是序列发送的方式，`combineLatest `是当组合前的任何一个序列发送新的信号时，它都会组合每一个`Observable `最新的元素，然后发送。
如果存在两条事件队列，需要同时监听，那么每当有新的事件发生的时候，combineLatest 会将每个队列的最新的一个元素进行合并。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/combinelatest.png)

```
example("combineLatest") {
    let disposeBag = DisposeBag()
    
    let stringSubject = PublishSubject<String>()
    let intSubject = PublishSubject<Int>()
    
    Observable.combineLatest(stringSubject, intSubject) { stringElement, intElement in
            "\(stringElement) \(intElement)"
        }
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
    
    stringSubject.onNext("🅰️")
    stringSubject.onNext("🅱️")
    intSubject.onNext(1)
    intSubject.onNext(2)
    stringSubject.onNext("🆎")
}
--- combineLatest example ---
🅱️ 1
🅱️ 2
🆎 2
```
### `switchLatest`
变换由`Observable`序列发出的元素融入到`Observable`序列,只发送最新插入到序列里面的元素。
 ![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/switch.png)
 
 ```
 example("switchLatest") {
    let disposeBag = DisposeBag()
    
    let subject1 = BehaviorSubject(value: "⚽️")
    let subject2 = BehaviorSubject(value: "🍎")
    
    let variable = Variable(subject1)
        
    variable.asObservable()
        .switchLatest()
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
    
    subject1.onNext("🏈")
    subject1.onNext("🏀")
    
    variable.value = subject2
    
    subject1.onNext("⚾️")
    
    subject2.onNext("🍐")
}
--- switchLatest example ---
⚽️
🏈
🏀
🍎
🍐
 ```
 
 在上面的例子中，在设置`variable.value`=`subject2`之后，向`subject1`中添加⚾️元素是没有效果的，因为最新插入到序列里面的`subject2`才会发送元素。
 
## Transforming Operators
我们可以对序列做一些转换，类似于swift中的`squence`转换。
### `map` 
对序列里面的每个元素进行转换。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/map.png)

```
example("map") {
    let disposeBag = DisposeBag()
    Observable.of(1, 2, 3)
        .map { $0 * $0 }
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- map example ---
1
4
9
```
### `flatMap`和`flatMapLatest`
在swift中，flatmap可以过滤nil及降维作用（二维数组转换为一维数组）。在RXSwift中，flatmap可以把序列中的每个元素转换为`Observable`序列，然后再把这组序列转换为新的`Observable`序列。比如你有一个`Observable`序列，然后这个序列里面的元素同样也可以作为`Observable`序列，那么久可以使用`flatMap`进行转换。`flatMap`和`flatMapLatest`的区别主要是`flatMapLatest`只会对最后一个加入序列里面的元素进行操作。
 ![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/flatmap.png)
 
 ```
 example("flatMap and flatMapLatest") {
    let disposeBag = DisposeBag()
    struct Player {
        var score: Variable<Int>
    }
    let 👦🏻 = Player(score: Variable(80))
    let 👧🏼 = Player(score: Variable(90))
    let player = Variable(👦🏻)
    
    player.asObservable()
        .flatMap { $0.score.asObservable() } // Change flatMap to flatMapLatest and observe change in printed output
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
    
    👦🏻.score.value = 85
    player.value = 👧🏼
    👦🏻.score.value = 95 // Will be printed when using flatMap, but will not be printed when using flatMapLatest
    👧🏼.score.value = 100
}
--- flatMap and flatMapLatest example ---
80
85
90
95
100
 ```
 `flatMapLatest`实际上是`map`和`switchLatest`的组合操作。
 
### `scan`
 
`scan`和swift里面的reduce操作类似，传入一个初始值，每次的结果作为下一次的输入值。最后返回把结果作为一个`Observable`元素。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/scan.png)

```
example("scan") {
    let disposeBag = DisposeBag()
    
    Observable.of(10, 100, 1000)
        .scan(1) { aggregateValue, newValue in
            aggregateValue + newValue
        }
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- scan example ---
11
111
1111
```

## Filtering and Conditional
下面这些操作可以对元素进行过滤。
### `filter`
和swift的操作一样，对元素进行过滤，返回符合闭包里面的元素。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/filter.png)

```
example("filter") {
    let disposeBag = DisposeBag()
    Observable.of(
        "🐱", "🐰", "🐶",
        "🐸", "🐱", "🐰",
        "🐹", "🐸", "🐱")
        .filter {
            $0 == "🐱"
        }
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- filter example ---
🐱
🐱
🐱
```
### `distinctUntilChanged`
禁止用`Observable`序列发送顺序重复元素,即他会抛弃相同的元素。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/distinct.png)

```
example("distinctUntilChanged") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐷", "🐱", "🐱", "🐱", "🐵", "🐱")
        .distinctUntilChanged()
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- distinctUntilChanged example ---
🐱
🐷
🐱
🐵
🐱
```
### `elementAt`
只发送指定index的元素。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/elementat.png)

```
example("elementAt") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .elementAt(3)
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- elementAt example ---
🐸
```
### `single`
只发送第一个元素或者第一个符合闭包条件的元素。

```
example("single with conditions") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .single { $0 == "🐸" }
        .subscribe { print($0) }
        .addDisposableTo(disposeBag)
    
    Observable.of("🐱", "🐰", "🐶", "🐱", "🐰", "🐶")
        .single { $0 == "🐰" }
        .subscribe { print($0) }
        .addDisposableTo(disposeBag)
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .single { $0 == "🔵" }
        .subscribe { print($0) }
        .addDisposableTo(disposeBag)
}
--- single with conditions example ---
next(🐸)
completed
next(🐰)
error(Sequence contains more than one element.)
error(Sequence doesn't contain any elements.)
```
### `take`
只发送指定个数的元素。
 ![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/take.png)

```
 example("take") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .take(3)
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- take example ---
🐱
🐰
🐶
```
 
### `takeLast`
同`take`只发送指定个数的元素，不同的是它从序列的最后开始。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/takelast.png)

```
example("takeLast") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .takeLast(3)
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- takeLast example ---
🐸
🐷
🐵
```

### `takeWhile`
发送元素一直到闭包里面的条件为false。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/takeuntil.png)

```
example("takeWhile") {
    let disposeBag = DisposeBag()
    
    Observable.of(1, 2, 3, 4, 5, 6)
        .takeWhile { $0 < 4 }
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- takeWhile example ---
1
2
3
```
### `takeUntil`
以一个序列为参考，当参考序列发送元素时，当前的序列则停止发送元素。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/takeuntil.png)

```
example("takeUntil") {
    let disposeBag = DisposeBag()
    let sourceSequence = PublishSubject<String>()
    let referenceSequence = PublishSubject<String>()
    
    sourceSequence
        .takeUntil(referenceSequence)
        .subscribe { print($0) }
        .addDisposableTo(disposeBag)
    
    sourceSequence.onNext("🐱")
    sourceSequence.onNext("🐰")
    sourceSequence.onNext("🐶")
    referenceSequence.onNext("🔴")
    sourceSequence.onNext("🐸")
    sourceSequence.onNext("🐷")
    sourceSequence.onNext("🐵")
}
--- takeUntil example ---
next(🐱)
next(🐰)
next(🐶)
completed
```
### `skip`
跳过发送指定个数的元素。从index为0开始计算。
 ![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/skip.png)

```
example("skip") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .skip(2)
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- skip example ---
🐶
🐸
🐷
🐵
```
### `skipWhile`
禁止符合闭包条件的元素发送。
 ![](http://reactivex.io/documentation/operators/images/skipWhile.c.png)
 
 ```
 example("skipWhile") {
    let disposeBag = DisposeBag()
    
    Observable.of(1, 2, 3, 4, 5, 6)
        .skipWhile { $0 < 4 }
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- skipWhile example ---
4
5
6
 ```
 同理还有`skipWhileWithIndex`和`skipUntil`（禁止发送元素知道参考的序列发送元素）。这里就不一一介绍了。
  
## Mathematical and Aggregate
下面几种方法主要是对序列进行集合操作。
### toArray
把可观察的序列转换为数组，然后把整个数组作为一个序列发送，最后终止。
![](http://reactivex.io/documentation/operators/images/to.c.png)

```
example("toArray") {
    let disposeBag = DisposeBag()
    
    Observable.range(start: 1, count: 10)
        .toArray()
        .subscribe { print($0) }
        .addDisposableTo(disposeBag)
}
--- toArray example ---
next([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
completed
```
### `reduce`
这个和swift中的使用方法一样，传入一个初始值，然后每次的结果作为下一次的传入值。最后的结果作为一个序列发送。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/reduce.png)

```
example("reduce") {
    let disposeBag = DisposeBag()
    
    Observable.of(10, 100, 1000)
        .reduce(1, accumulator: +)
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
1111
```
### `concat`
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/concat.png)

```
example("concat") {
    let disposeBag = DisposeBag()
    
    let subject1 = BehaviorSubject(value: "🍎")
    let subject2 = BehaviorSubject(value: "🐶")
    let variable = Variable(subject1)
    
    variable.asObservable()
        .concat()
        .subscribe { print($0) }
        .addDisposableTo(disposeBag)
    
    subject1.onNext("🍐")
    subject1.onNext("🍊")
    variable.value = subject2
    subject2.onNext("I would be ignored")
    subject2.onNext("🐱")
    subject1.onCompleted()
    subject2.onNext("🐭")
}
```

## Error Handling
下面几个操作可以捕获异常事件，并且可以从异常出恢复。
### `catchErrorJustReturn`
捕获异常事件，并从异常处恢复，然后发送一个元素，最后终止。
![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/catch.png)

```
example("catchErrorJustReturn") {
    let disposeBag = DisposeBag()
    let sequenceThatFails = PublishSubject<String>()
    
    sequenceThatFails
        .catchErrorJustReturn("😊")
        .subscribe { print($0) }
        .addDisposableTo(disposeBag)
        
    sequenceThatFails.onNext("😡")
    sequenceThatFails.onNext("🔴")
    sequenceThatFails.onError(TestError.test)
}
--- catchErrorJustReturn example ---
next(😡)
next(🔴)
next(😊)
completed
```
### `catchError`
捕获异常事件，并从异常处恢复，返回指定的事件序列。
 ![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/catch.png)
 
 ```
 example("catchError") {
    let disposeBag = DisposeBag()
    let sequenceThatFails = PublishSubject<String>()
    let recoverySequence = PublishSubject<String>()
    
    sequenceThatFails
        .catchError {
            print("Error:", $0)
            return recoverySequence
        }
        .subscribe { print($0) }
        .addDisposableTo(disposeBag)
    
    sequenceThatFails.onNext("😡")
    sequenceThatFails.onNext("🔴")
    sequenceThatFails.onError(TestError.test)
    recoverySequence.onNext("😊")
}
--- catchError example ---
next(😡)
next(🔴)
Error: test
next(😊)
 ```
 ### `retry`
 重新尝试，意思就是当异常发生的时候，从头开始发送事件。
  ![](https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/MarbleDiagrams/png/retry.png)
  
  ```
  example("retry") {
    let disposeBag = DisposeBag()
    var count = 1
    
    let sequenceThatErrors = Observable<String>.create { observer in
        observer.onNext("🍎")
        observer.onNext("🍐")
        observer.onNext("🍊")
        
        if count == 1 {
            observer.onError(TestError.test)
            print("Error encountered")
            count += 1
        }
        
        observer.onNext("🐶")
        observer.onNext("🐱")
        observer.onNext("🐭")
        observer.onCompleted()
        
        return Disposables.create()
    }
    
    sequenceThatErrors
        .retry()
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- retry example ---
🍎
🍐
🍊
Error encountered
🍎
🍐
🍊
🐶
🐱
🐭
  ```
  
## Debugging Operators
主要讲调试RX代码的方法。
### `debug`
通过这个方法可以在发生错误的时候打印出所有的订阅者，事件等。

```
example("debug") {
    let disposeBag = DisposeBag()
    var count = 1
    
    let sequenceThatErrors = Observable<String>.create { observer in
        observer.onNext("🍎")
        observer.onNext("🍐")
        observer.onNext("🍊")
        
        if count < 5 {
            observer.onError(TestError.test)
            print("Error encountered")
            count += 1
        }
        
        observer.onNext("🐶")
        observer.onNext("🐱")
        observer.onNext("🐭")
        observer.onCompleted()
        
        return Disposables.create()
    }
    
    sequenceThatErrors
        .retry(3)
        .debug()
        .subscribe(onNext: { print($0) })
        .addDisposableTo(disposeBag)
}
--- debug example ---
2016-09-24 22:25:47.466: playground73.swift:42 (__lldb_expr_73) -> subscribed
2016-09-24 22:25:47.468: playground73.swift:42 (__lldb_expr_73) -> Event next(🍎)
🍎
2016-09-24 22:25:47.469: playground73.swift:42 (__lldb_expr_73) -> Event next(🍐)
🍐
2016-09-24 22:25:47.469: playground73.swift:42 (__lldb_expr_73) -> Event next(🍊)
🍊
Error encountered
2016-09-24 22:25:47.472: playground73.swift:42 (__lldb_expr_73) -> Event next(🍎)
🍎
2016-09-24 22:25:47.472: playground73.swift:42 (__lldb_expr_73) -> Event next(🍐)
🍐
2016-09-24 22:25:47.472: playground73.swift:42 (__lldb_expr_73) -> Event next(🍊)
🍊
Error encountered
2016-09-24 22:25:47.474: playground73.swift:42 (__lldb_expr_73) -> Event next(🍎)
🍎
2016-09-24 22:25:47.474: playground73.swift:42 (__lldb_expr_73) -> Event next(🍐)
🍐
2016-09-24 22:25:47.475: playground73.swift:42 (__lldb_expr_73) -> Event next(🍊)
🍊
Error encountered
2016-09-24 22:25:47.477: playground73.swift:42 (__lldb_expr_73) -> Event error(test)
Received unhandled error: /var/folders/49/3_69hmqj64j4g5p0pz56w72h0000gn/T/./lldb/25555/playground73.swift:43:__lldb_expr_73 -> test
2016-09-24 22:25:47.491: playground73.swift:42 (__lldb_expr_73) -> isDisposed
2016-09-24 22:25:47.491: playground73.swift:42 (__lldb_expr_73) -> isDisposed
```

### `RxSwift.resourceCount`
RXSwift还提供了查看当前穿件的所有RX资源个数的方法，通过这个方法可以有效的调试泄露问题。

```
example("RxSwift.resourceCount") {
    print(RxSwift.resourceCount)
    let disposeBag = DisposeBag()
    print(RxSwift.resourceCount)
    let variable = Variable("🍎")
    let subscription1 = variable.asObservable().subscribe(onNext: { print($0) })
    
    print(RxSwift.resourceCount)
    
    let subscription2 = variable.asObservable().subscribe(onNext: { print($0) })
    
    print(RxSwift.resourceCount)
    subscription1.dispose()
    print(RxSwift.resourceCount)
    subscription2.dispose()
    print(RxSwift.resourceCount)
}
print(RxSwift.resourceCount)
--- RxSwift.resourceCount example ---
0
1
🍎
4
🍎
6
5
4
0
```
`RxSwift.resourceCount`并不是默认开启的，在release模式下请不要开启它。通过下面2种方法可以在开发的时候开启：

```
**CocoaPods**
 1. Add a `post_install` hook to your Podfile, e.g.:
 
 target 'AppTarget' do
 pod 'RxSwift'
 end
 
 post_install do |installer|
     installer.pods_project.targets.each do |target|
         if target.name == 'RxSwift'
             target.build_configurations.each do |config|
                 if config.name == 'Debug'
                     config.build_settings['OTHER_SWIFT_FLAGS'] ||= ['-D', 'TRACE_RESOURCES']
                 end
             end
         end
     end
 end
 
 2. Run `pod update`.
 3. Build project (**Product** → **Build**).
 #
 **Carthage**
 1. Run `carthage build --configuration Debug`.
 2. Build project (**Product** → **Build**).
```


