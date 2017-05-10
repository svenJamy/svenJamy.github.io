---
title: swift Sequence及map，flatmap的实现
date: 2016-09-20 09:40:16
tags: swift
---

## swift Sequence

Sequence和IteratorProtocol两个协议提供连续迭代访问元素的方法。Swift 的 for...in 可以用在所有实现了 Sequence 的类型上。<!-- more --> swift的Sequence默认实现了一些方法，如contains，sorted，filter等，更多定义请查看文档：![http://swiftdoc.org/v3.0/protocol/Sequence](http://swiftdoc.org/v3.0/protocol/Sequence)

Sequence协议默认实现了```makeIterator()```和Iterator的```next()```方法，我们可以直接使用默认的方法：

```swift
extension Sequence {
  func reduce1(combine:(Iterator.Element, Iterator.Element) -> Iterator.Element) -> Iterator.Element? {
    var i = makeIterator()
    guard var first = i.next() else {
      return nil
    }
    
    while let element = i.next() {
      first = combine(first, element)
    }
    return first
  }
}

let animals = ["Antelope", "Butterfly", "Camel", "Dolphin"]
animals.reduce1 { (current, element) -> String in
  if current.characters.count > element.characters.count {
    return current
  } else {
    return element
  }
}
// print（"Butterfly"）
```

当然我们也可以自己定义一个实现了Sequence协议的类型，比如在使用for...in，contains方法的时候，我们希望使用自己实现的迭代策略。Sequence协议需要实现一个```makeIterator()```方法，这个方法返回一个iterator，iterator是实现了IteratorProtocol协议的类型：

```swift
struct Countdown: Sequence, IteratorProtocol {
  let start: Int
  var times = 0
  
  init(start: Int) {
    self.start = start
  }
  
  mutating func next() -> Int? {
    let nextNumber = self.start - times
    guard nextNumber > 0
      else { return nil }
    
    times += 1
    return nextNumber
  }
}

let threeTwoOne = Countdown(start: 3)
for count in threeTwoOne {
  print("\(count)...")
}
```

### map和flatmap
#### map

map方法接受一个闭包作为参数，然后遍历整个Sequences，对里面的每一个元素执行闭包里面的操作，相当于做一个映射。方法返回一个数组。我们可以看看方法的定义和实现：

```
func map<T>(_ transform: @noescape (Self.Iterator.Element) throws -> T) rethrows -> [T]
```
去除@noescape rethrows等和实现逻辑无关的修饰符，我们再来看看它的实现：

```
  public func map<T>(_ transform: (Iterator.Element) -> T) -> [T] {
    let count: Int = numericCast(self.count)
    // 取得数组的个数，为0的话执行方法空
    if count == 0 {
      return []
    }
    // 定义一个存放结果的数组
    var result = ContiguousArray<T>()
    result.reserveCapacity(count)

    var i = self.startIndex
	 // 关键代码，对数组里面的每一个元素执行闭包操作,
    for _ in 0..<count {
      result.append(try transform(self[i]))
      formIndex(after: &i)
    }

    _expectEnd(i, self)
    return Array(result)
  }
```
```transform: (Iterator.Element) -> T```闭包的参数类型是当前数组里面的元素类型，但是返回值可以是任意的。
map的使用在日常工作中还是很常见的，使用方法也不是很复杂，下面我们来看看flatmap.
#### flatmap
flatmap有两个重载，意思就是说使用这个方法可以有两种不同的作用，下面我们对他们一一分析：
> 1:

我们先来看看第一个例子：flatmap操作对每个元素执行transform闭包，返回一个非空的数组。使用flatmap方法可以把一个option的数组转换为一个非option的数组：

```
let possibleNumbers = ["1", "2", "three", "///4///", "5"] 
let mapped: [Int?] = numbers.map { str in Int(str) }
// [1, 2, nil, nil, 5]
let flatMapped: [Int] = numbers.flatMap { str in Int(str) }
// [1, 2, 5]
```
源码实现（源码路径： swift/stdlib/public/core/SequenceAlgorithms.swift.gyb或者swift/stdlib/public/core/FlatMap.swift）：

```swift
public func flatMap<ElementOfResult>(_ transform: (${GElement}) throws -> ElementOfResult?) rethrows -> [ElementOfResult] {
    var result: [ElementOfResult] = []
    for element in self {
      // 执行闭包操作，如果值不为空的话则添加到返回结果数组中
      if let newElement = try transform(element) {
        result.append(newElement)
      }
    }
    return result
  }
```

> 2

再看看第二个例子：返回一个非option数组，数组里面的新元素是原来每个元素执行闭包后的结果的串联。

```
let numbers = [1, 2, 3, 4]
let mapped = numbers.map { Array(count: $0, repeatedValue: $0) }
// [[1], [2, 2], [3, 3, 3], [4, 4, 4, 4]]
 
let flatMapped = numbers.flatMap { Array(count: $0, repeatedValue: $0) }
// [1, 2, 2, 3, 3, 3, 4, 4, 4, 4]
```

源码实现：

```
public func flatMap<SegmentOfResult : Sequence>(
    _ transform: (${GElement}) throws -> SegmentOfResult
  ) rethrows -> [SegmentOfResult.${GElement}] {
    var result: [SegmentOfResult.${GElement}] = []
    for element in self {
      result.append(contentsOf: try transform(element))
    }
    return result
  }
```
从上面的两个源码中我们看出其实内部的实现很简单，就是把符合闭包的元素加入到新的数组中，唯一不同的就是两个append操作：
第一个是```append(newElement)```这个方法直接将元素加入到数组中,第二个是```append(contentsOf: try transform(element))```这个方法是将集合里面的所有元素一一的添加到新的数组中所以就有了```[1,2],[3,4]->[1,2,3,4]```。

第二个实现其实和```Array(s.map(transform).joined())```是一样的，对map后的数组进行```joined```操作。



文档更新中，写的不足的地方请谅解，您也可以留下您宝贵的意见，谢谢~~~





