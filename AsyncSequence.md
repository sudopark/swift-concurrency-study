## AsyncSequence

https://developer.apple.com/documentation/swift/asyncsequence

: 원소를 비동기, 순차 반복적으로 접근할 수 있는 타입



### Overview

AsyncSequence는 리스트의 엘리먼트를 한번에 하나씩 접근할 수 있는 Sequece 타입에 비동기성을 추가한것과 같습니다. AsyncSequece를 처음 이용하려고 할때 그것들의 원소는 전부 또는 일부만 혹은 아예 없을 수 있습니다. 대신 값이 준비될때 받기위해  await를 이용할 수 있습니다.

Sequence로써 await-in loop를 이용하여 이를 순회할 수 있습니다. 하지만 호출자는 값을 기다려야만 하기 때문에 await 키워드를 사용하여야만 합니다.  아래의 예제는 1부터 howHigh까지 값을 생산해내는 커스텀 AsyncSequence Counter의 값들을 순회화는 방법을 나타냅니다.

```swift
for await i in Counter(howHigh: 10) {
  print(i, terminator: " ")
}
// Prints: 1 2 3 4 5 6 7 8 9 10
```

AsyncSequence는 값을 생산해내거나 포함하지 않습니다. 이는 단지 값들에 어떻게 접근해야하는지만 나타냅니다. Element라 불리는 associated type 값과함께 AsyncSequence는 makeAsyncIterator() 메소드를 정의하고 이는 AsyncIterator 인스턴스를 반환합니다. 일반적인 IteratorProtocol과 같이 AsyncIteratorProtocol은 오직 엘리먼트를 발생시키기 위해  next() 메소드가 필요로 합니다. AsyncIterator에서 다른점은 next() 메소드가 비동기로 정의된다는 것 입니다. 그리고 이로인해 호출자는 await 키워드와 함께 next()의 반환을 기다려야합니다.

AsyncSequence는 또한 기본 Sequence에서 제공하는 작업을 모델링하여 값을 처리하기 위한 메소드들을 정의합니다. 이 메소드들은 크게 두가지 종류로 나뉩니다: 단일 값을 반환하거나, 또다른 AsyncSequence를 반환하는 경우 말입니다.

단일값을 반환하는 경우에는 값을 얻기 위해 await-in loop가 필요 없습니다. 대신에 await 호출을 하면 됩니다. 예를들어 contains(_:) 메소드는 AsyncSequece내 주어진 값이 존재하는지를 나타내는 Bool 값을 반환합니다. 위의 카운터 예제에서 다음과 같이 테스트ㅐㅎ볼수 있습니다.

```swift
let found = await Counter(howHigh: 10).contains(5) // true
```

다른 AsyncSequence를 리턴하는 종류의 메소드들은 메소드의 의미에 특정한 유형을 리턴합니다.(Methods that return another `AsyncSequence` return a type specific to the method’s semantics. ) 예를들어 map(_:) 메소드는 AsyncMapSequence를 반환합니다. ( map(:_)에 제공되는 클로저가 에러를 스로우한다면 AsyncThrowingMapSequence가 반환됩니다.) 반환된 시퀀스는 시퀀스의 다음 멤버를 기다릴 필요는 없습니다. 이는 단지 호출자가 언제 작업을 시작할지에 따라 결정됩니다. 대부분 이경우의 시퀀스들은 처음에 예제에서 보았듯이 for await-in를 이용하여 순회할것입니다. 아래의 예제에서 map(_:) 메소드는 Counter에서 받은 Int 값을 String으로 변환합니다.

```swift
let stream = Counter(howHigh: 10)
	.map { $0 % 2 == 0 ? "Even" : "Odd" }
for await s in stream {
  print(s, terminator: " ")
}
// Prints: Odd Even Odd Even Odd Even Odd Even Odd Even
```

-----

## AsyncIteratorProtocol

https://developer.apple.com/documentation/swift/asynciteratorprotocol

: 한번에 하나씩 시퀀스의 값을 비동기적으로 제공하는 타입



### Overview

AsyncIteratorProtocol은 AsyncSequence의 makeAsyncIterator()의 반환하는 타입을 정의합니다. 간단히 말해 iterator는 비동기 시퀀스의 값을 생산합니다. 이 프로토콜은 오직 하나의 메소드만 정의하는데 비동기 next() 메소드 입니다. 이는 시퀀스의 다름 값을 비동기적으로 반환하거나 nil을 반환하여 시퀀스의 끝임을 알립니다.

AsyncSequence를 직접 만들기 위해서는 AsyncIterator을 준수하는 래핑된 타입을 구현하세요. 다음 예제는 내부적으로 howHigh에 도달할때까지 반복적으로 Int 값을 만들어내는 카운터입니다. 이 예제에서 실제 로직은 비동기적이지 않지만 어떻게 Sequence와 Interator를 정의해야할지 대략적으로 알수있습니다.

```swift
struct Counter: AsyncSequence {
  typealias Element = Int
  let howHigh: Int
  
  struct AsyncIterator: AsyncInteratorProtocol {
    let howHigh: Int
    var current = 1
    
    mutating func next() async -> Int? {
      // 진짜 비동기로 구현이라면 'Task' API를 이용하여 이 지점에서 취소를 확인하고 종료하거나 합니다.
      guard current <= howHigh else {
        return nil
      }
      let result = current
      current += 1
      return result
    }
  }
  
  func makeAsyncIterator() -> AsyncIterator {
    return AsyncIterator(howHigh: howHigh)
  }
}

// 호출은 다음과같이 합니다.
for await i in Counter(howHigh: 10) {
  print(i, terminator: " ")
}
// Prints: 1 2 3 4 5 6 7 8 9 10
```

### End of Iteration

iterator가 nil을 반환하면 시퀀스가 종료되었음을 뜻합니다. next()에서 nil을 반환하거나(혹은 error 를 throw 하거나) iterator는 종료 상태로 진입하고 후에 호출되는 next()들은 다 nil이 반환됩니다.

### Cancellation

AsyncIteractorProtocol을 준수하는 타입은 취소시에 Swift의 Tak API에서 제공하는 기능을 사용해야합니다. Iteractor는 취소를 핸들링하고 응답할지 방식을 다음 포함하여 선택할 수 있습니다.

- next( ) 안에서 현재 Task의 isCancelled 값을 참조하여 시퀀스를 종료시키기위해 nil을 반환해야 합니다.
- CancellationError를 던지는 경우 Task에서 checkCancellation( )을 호출해봅니다.
- 취소에 즉시 반응하기 위하여 next( )를 wiuhTaskCancellationHandler(handler: operation:)과 함께 구현합니다.

-----

## AsyncStream

https://developer.apple.com/documentation/swift/asyncstream/

: 새 엘리먼트를 만들어내는 continuation을 호출하는 클로저를 이용해 새 비동기 시퀀스를 만듭니다.



### Overview

AsyncStream AsyncSequence 프로토콜을 따르고 async iterator를 정의할 필요없이 손쉽게 비동기 시퀀스를 만들 수 있습니다. AsyncStream은 async-await와 함께하는 콜백이나 델리게이션 API에 잘 맞습니다.

AsyncStream.Continuation을 받는 클로저를 이용하여 비동기 스트림을 쉽게 만들 수 있습니다. 클로저 내에서 값을 만들어 내고 continuation의 yield(_:) 메소드를 이용하여 이를 스트림에 제공하세요. 더이상 만들어낼 값이 없다면 finish( ) 메소드를 호출하세요. 이를 통하여 sequence iterator는 nil을 생산해내고 시퀀스를 종료 시킵니다. continuation은 AsyncStream 외부의 동기 context에서 호출을 허용하는 Sendable 프로토콜을 따릅니다.

엘리먼트의 임의 소스는 iterating하는 호출자가 소비하는것보다 더 빠르게 값을 만들어낼 수 있습니다. 이로인해 AsyncStream은 버퍼링 동작을 정의하여 스트림이 특정수의 가장 오래된 혹은 최신의 엘리먼트들을 버퍼링 할 수 있도록 합니다. 기본적으로 버퍼 사이즈의 한계는 Int.max 까지입니다.(한계가 없음)

### Adapting Existing Code to Use Stream

기존의 콜백 코드를 async-await 구문을 만족시키게 하기위해, continuation의 yeild(_:)의 메소드를 이용하여 콜백이 만들어내는 값을 스트림에 제공하세요.

지진이 날때마다 Quake 인스턴스를 반환하는 가상의 QuakeMonitor 라는 타입이 있다 가정해봅시다. 콜백을 받기위해서 호출자는 QuakeMonitor의 quakeHandler를 등록합니다.

```swift
class QuakeMonitor {
  var quakeHandler: ((Quake) -> Void)?
  
  func staftMonitoring() { ... }
  func stopMonitoring() { ... }
}
```

async-await에 적용시키기위해 QuakeMonitor에 AsyncStream<Quake> 타입의 quakes 프로퍼티를 추가하여 기능을 확장하세요. 이 프로퍼티의 게터에서 AsyncStream을 반환하기위한 이의 빌드 클로져에서(런타임에서 호출되고 스트림을 만드는) 다음의 절차를 따라 continuation을 이용하세요.

1. QuakeMonitor 인스턴스를 만드세요
2. Quake 인스턴스를 수신하기위해 quakeHandler를 등록하세요. 그리고 이를 ocntinuation의 yield(_:) 메소드를 이용하여 포워딩 하세요.
3. continuation의 onTermination 클로저 프로퍼티를 세팅해 호출되면 모니터의 stopMonitoring( )을 호출하세요.
4. 모니터의 startMonitoring을 호출하세요

```swift
extension QuakeMonitor {
  static var quakes: AsyncStream<Quake> {
    AsyncStream { continuation in
      let monitor = QuakeMonitor()
      monitor.quakeHandler = { quake in
        continuation.yeild(quake)
      }
      continuation.onTermination = { @Sendable _ in
        monitor.stopMonitoring()
      }
      monitor.startMonitoring()
    }
  }
}
```

stream은 AsyncSequence이기때문에 호출지점에서는 for-await-in 구문을 이용하여 스트림이 만들어내는 Quake 인스턴스를 처리할 수 있습니다.

```swift
for await quake in QuakeMonitor.quakes {
  print("Quake: \(quake.date)")
}
print("Stream finished")
```



#### Sendable 

https://developer.apple.com/documentation/swift/sendable

: Sendable 프로토콜은 해당 타입의 값이 동시성 코드에서도 안전하게 쓰일 수 있음을 나타냅니다.

(Actor 프로토콜이 Sendable 프로코콜을 상속함) -> Actor는 thread safe한 클래스, Sendable은 더 상위 개념의 thread safe한 익명 함수도 포함



#### AsyncStream.Continuation

https://developer.apple.com/documentation/swift/asyncstream/continuation

동기코드와 비동기 스트림간의 상호작용을 위한 인터페이스

##### Declaration

```swift
struct AsyncStream<Element>.Continuation 
```

##### OverView

AsyncStream의 생성자에서 제공하는 클로저는 Continuation의 인스턴스를 호출시점에 받습니다. 이 continuation을 이용해 스트림에 엘리먼트를 제공하세요(yield 메소드를 이용하여) 그리고 finish( ) 메소드를 이용하여 스트림을 종료시키세요



더 참고할부분: https://github.com/apple/swift-evolution/blob/main/proposals/0314-async-stream.md 

------

## RxSwift 6.5.0 - Swift Concurrency is Here

https://github.com/ReactiveX/RxSwift/blob/main/Documentation/SwiftConcurrency.md

RxSwift 6.5 부터 Obsrvable과 다른 reactive 유닛들이 마치 비동기 작업이나 시퀀스 인것처럼  await를 사용할 수 있습니다. 또한 async 작업을 Observable로 변환할 수 있습니다.



#### awaiting values emitted by Observable

Observable에서 방출되는 값을 기다리는 방식은 다음 3가지 변수가 있습니다. - 방출하는 값의 특성 수에 따라, 에러를 스로우 하거나 말거나

세가지 유형은 다음과 같습니다: 시쿼스를 기다리거나, 에러를 던지지 않는 시퀀스를 기다리거나, 단일 값을 기다리거나

##### Awaiting a following sequence

일반적으로 Observable으 에러를 방출하고, 마찬가지로 async/await 세계에서도 에러를 스로우합니다.

Observable의 라이프사이클 내에서 전체 요소에 대해 다음과 같이 순회할수있습니다.

```swift
do {
  for try await value in observable.values {
    print("Got a value: \(value)")
  }
} catch {
  print("Got an error: \(error)")
}
```

Observable이 무조건 종료되어야한다는점에 주목하세요. 아니라면 그렇지 않다면 비동기 테스크는 중단되지않고 상위 작업도 다시 시작될 수없습니다.

##### Awaiting a non-throwing sequence

Infalliable, Driver, Signal은 에러를 방출하지 않을것임이 보장됩니다(observable과 반대로). 그렇기에 error를 catch하는것을 신경쓸 필요없이 값들을 순회할 수 있습니다.

```swift
for await value in observable.values {
  print("Got a value: \(value)")
}
```

##### Awaiting a single value

무한한 시퀀스가 가능한 위와 달리, primitive sequence들은 하나도 혹은 하나의 값을 방출할것임이 보장됩니다. 이 경우에서 다음과 같이 간단하게 값들을 기다릴 수 있습니다.

```swift
let value1 = try await sinle.value // Element
let value2 = try await maybe.value // Element? -> maybe 가 값방출 없이 끝난다면 nil이 반환됨
let value3 = try await completable.value // Void -> 종료되었음을 나타내기위해 Void만 반환됨
```

#### Wrapping an async Task as an Observable

 이미 AsyncSequence가 이미 있는경우(AsyncStream과 같이) 이를 asObservable()을 이용하여 다음과같이 rx 형시으로 이용할 수 있습니다.

```swift
let stream = AsyncStream { ... }
stream.asObservable()
	.subscribe(
     onNext: { ... },
	   onError: { ... }
  )
```



##### [구현코드 예제: Observable+Concurrency](https://github.com/ReactiveX/RxSwift/blob/main/RxSwift/Observable%2BConcurrency.swift)

```swift
public extension ObservableConvertibleType {
    /// Allows iterating over the values of an Observable
    /// asynchronously via Swift's concurrency features (`async/await`)
    ///
    /// A sample usage would look like so:
    ///
    /// ```swift
    /// do {
    ///     for try await value in observable.values {
    ///         // Handle emitted values
    ///     }
    /// } catch {
    ///     // Handle error
    /// }
    /// ```
    var values: AsyncThrowingStream<Element, Error> {
        AsyncThrowingStream<Element, Error> { continuation in
            let disposable = asObservable().subscribe(
                onNext: { value in continuation.yield(value) },    // value 스트림으로 전달
                onError: { error in continuation.finish(throwing: error) },  // error 전달하며 종료
                onCompleted: { continuation.finish() },     // 스트림 으로 종료되었음을 전달
                onDisposed: { continuation.onTermination?(.cancelled) }  // dispose 되었을때 continuation의 onTermination 호출해줌
            )

            continuation.onTermination = { @Sendable _ in
                disposable.dispose()   // onTermination 불리면 dispose -> continuation 살아있을때까지 구독 지속
            }
        }
    }
}

@available(macOS 10.15, iOS 13.0, watchOS 6.0, tvOS 13.0, *)
public extension AsyncSequence {
    /// Convert an `AsyncSequence` to an `Observable` emitting
    /// values of the asynchronous sequence's type
    ///
    /// - returns: An `Observable` of the async sequence's type
    func asObservable() -> Observable<Element> {
        Observable.create { observer in
            let task = Task {
                do {
                    for try await value in self {
                        observer.onNext(value)
                    }

                    observer.onCompleted()
                } catch {
                    observer.onError(error)
                }
            }

            return Disposables.create { task.cancel() }		// dispose 되면 task cancel -> observable 구독 살아있을때까지 task 생존
        }
    }
}
```

