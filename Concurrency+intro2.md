

## Concurrency
출처: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/
swift는 구조적인 방식으로 비동기-병렬 코드를 지원합니다. 비동기-병렬처리를 스케쥴링하는것은 복잡도를 증가시킵니다. swift는 컴파일 타임 체크를 이용하여 개발자의 의도를 드러낼 수 있게합니다. 

#### Note
swift가 제공하는 concurrency 모듈은 스레드 위에서 돌아가지만, 스레드를 직접적으로 다룰필요없게합니다. Swift의 비동기 함수는 실행중인 스레드를 양보?하고 다른 비동기작업이 첫번째 작업이 블럭되는 동안 실행 가능하게 합니다. 비동기 함수가 다시 시작되었을때 어떤 스레드에서 실행될것인지는 장담할 수 없습니다.

// 구문 및 사용법

### 비동기 함수를 정의하고 호출하는 방법
 -비동기 함수 혹은 메소드란 실행 도중에 일시중지(suspended)될 수 이쓴ㄴ 특별한 종류의 함수 or 메소드 입니다. 이들은 값을 반환, throw error 혹은 아무것도 리턴하지 않는 동작을 하더라도 중간에 일시중지 될 수 있습니다. body 구분 내에서 이와같이 일시중지될수이쓴ㄴ 부분을 표시해야합니다.

 ```swift
 func listPhoto(inGallery name: String) async -> [String] {
 	...
 }
 // error return 하는 경우에는 async throw
 ```

 비동기 메소드를 호출하는 경우에 해당 메소드가 값을 반환하기 전까지 실행이 일시중지됩니다. 이와같은 일시중지 지점에대해 호출 이전에 await 키워드를 붙여야 합니다. 
 비동기 메소드 내에서 오직 다른 비동기 작업을 호출하는 경우에만 실행이 일시중지됩니다. - suspension은 절대로 암묵적이거나 선점적이지 않습니다 -> 가능한 suspension 지점에는 await 로 표시되어야합니다.

 ```swift
 let photoNames = await listPhotos(inGallery: "some")	// suspension
 let sortedNames = photoNames.sorted()
 let name = sortedNames[0]
 let photo = await downloadPhoto(named: name)		// suspension
 show(photo)

 ```
 위의 코드 실행 예제
 1. 첫번째 suspension 포인트를 만나고 listPhotos에서 값이 반환되기 전까지 실행이 일시중지됨
 2. 위의 코드 실행이 중지된동안 다른 concurrent code가 실행됩니다.
 3. listPhotos 값이 반환되면 이전 중지 지점부터 다시 시작되고 -> photoNames에 값이 할당됩니다.
 4. sortedNames, name이 있는 줄의 코드에는 suspension 포인트가 없고 다른 동기 코드들과 같이 실행됩니다.
 5. downloadPhoto에서 또한번 값이 반환될때까지 실행이 일시중지됩니다. 

 위에서 await로 표시된 suspension 포인트는 코드가 일시중지되고 값이 반환될때까지 기다릴 수 있음을 의미합니다. 이른 다른 이름으 스레드를 양보?(yielding the thread)라고 합니다 -> 기저에서 swift는 실행중이던 스레드에서 해당 코드를 일시중지 시키고 다른 코드를 실행시킬 수 있기 때문입니다. 

 await가 있는 코그는 실행을 일시중지시킬 수 있어야하기때문에 확실한 경우에만 이들을 호출할 수 있습니다.
 - 비동기 함수, 메소드 혹은 프로퍼티의 body 내부에 코드가 있는 경우
 - @main이라 마킹된 struct, class or enumeration의 static main() 메소드 내에서
 - unstructured child task 내부에서


 ### Asynchronous Sequence
 위의 listPhotos(inGallery:) 예제에서 결과값 어레이를 한번에 반환하는것과 다르게 asynchronous sequence를 사용하여 collection의 element를 각자 기다리는 방식이 있습니다.
 ```swift
 let handle = FileHandle.standardInput
 for try await line in handle.bytes.lines {
 	print(line)
 }
 ```
 for-await-in loop는 iteration 중에 일시중지될 수 있음을 나타냅니다. -> 다음 엘리먼트가 준비될때까지

 sequence protoocol을 체택하게하여 for-in loop를 사용하듯이 특정 타입이 AsyncSequence protocol을 따륵 하여 for-await-in loop를 사용하게 할 수 있습니다.


 ### Calling Asynchronous functions in parallel
 아래의 코드는 앞선 await 구문이 끝나기 전까지 실행이 시작되지 않습니다.
 ```swift
 let firstPhoto = await downloadPhoto(names: "some")
 let secondPhoto = await downloadPhoto(names: "some")
 let thirdPhoto = await downloadPhoto(names: "some")

 let photos = [firstPhoto, secondPhoto, thirdPhoto]
 show(photos)
 ```
위의 케이스는 download 동작이 순차적으로 진행될 필요가 없고 위와같이 로직을 작성하는 것은 concurrency의 장점을 살리지 못하고 비효율적이게 됩니다.

비동기 함수들을 병렬적으로 실행되도록 하기 위해서 constant를 선언하는 let 이전에 async를 붙이고 이 상수값이 필요할때에 await를 거세요

```swift
async let firstPhoto = ...
async let secondPhoto = ...
async let thirdPhoto = ...

let photos = await [firstPhoto, secondPhoto, thirdPhoto]
show(photos)
```

위 두 다른 사용법에 대한 정리
- 다음 코드 구문이 이전 비동기 코드의 결과를 필요로 하는 경우에 await를 붙여라 -> 순차적으로 결과가 전달되어야하는 경우
- 위와같지 않다면 async-let 구문을 사용해랴
- await나 async-let 둘다 이들이 suspend된 동안에 다른 코드들이 실행될 수 있게 해준다.
- 두경우 모두 await가 표시된 부분은 필요시 실행이 중지되고 값이 반환될때까지 대기됨을 의미합니다.

--------

// 구현

## Tasks and Task Groups
Task는 비동기 작업이 실해오디는 단위 입니다. async-let 구문은 child task를 만들어 냅니다. 아니면 task group을 만들고 여기에 child task를 넣을 수 있습니다. 이를 통해 우선순위를 조정하거나 취소를 제어할 수 있습니다.
task group내 존재하는 task들은 공통의 parent task를 지니고 각기 별도의 child task를 지닐 수 있습니다. 이와 같은 task와 task group간의 관계로 인하여 이를 structured concurrency라 부릅니다. 이러한 명백한 parent-child 간의 관계로 인해 swift가 자체적으로 취소와 같은 일들을 처리할 수 있고, 컴파일타임에서도 오류를 잡아낼 수 있습니다.
```swift
await withTaskGroup(of: Data.self) { taskGroup in
	let photoNames = await listPhotos(inGallery: "soome")
	for name in photoNames {
		taskGroup.addTask { await downloadPhoto(named: name) }
	}
}
```

#### Task
task group을 만들기 위해서 withTaskGroup(of:returning:body:) method를 사용하세요.
또한 이겨경우에 taskGroup을 생성클로저 외부에서 사용하지 마세요 -> 이는 mutation operation에 보호받지 못하기에 일단 Swift type system에 의해 불가능하기는 할거임

#### Task execution order
task group에 추가된 task는 concurrent 하게 실행되거나, 순서에 따라 스케쥴링될 수 있음

#### Cancellation behavior
아래의 방법을 통하여 task group은 취소될 수 있음
- cancellAll() 메소드가 호출되거나
- 해당 task group을 실행하는 Task가 취소되거나
-> TaskGroup의 구조화된 특성 때문에 cancellation은 하위 child-task 들에게 전파됩니다.

취소된 task group에도 task가 추가될 수 있지만, 이들은 실행되고 바로 취소될꺼임 -> 이를 방지하기 위해 taskGroup에 task를 추가할 때, addTask 보다는 addTaskUnlessCancelled(priority:body:)를 사용하세요

### Unstructured Concurrency
swift는 구조화되지 않은 concurrency를 지원하기도 합니다.
- Task.init(priority:operation:) -> 현재 actor에서 실행되는 unstructured task를 생성
- Task.detached(priority:operation:) -> 현재 actor와 관련없이 실행되는 unstructured task 생성

```swift
let newPhoto = ...
let handle = Task {
	return await add(newPhoto, toGalleryNamed: "some")
}
let result = await handle.value
```

### Task Cancellation
swift concurrency는 coorperatice cancellation model을 사용합니다. 모든 Task는 실행중 적절한 순간에 자신이 취소 되었는지 체크합니다. 그리고 그런 경우 적절한 조취를 취해야하는데 보통 아래와 같은 방법을 사용합니다.
- CancellationError와 같은 error를 throw
- nil이나 빈 컬렉션을 반환
- 일부만 완성된 결과물을 반환
최소되었음을 판단하는 경우에는 아래의 방법을 사용합니다.
- Task.checkCancellation() -> 만약 취소되었면 CancellationError를 throw
- Task.isCanncelled 
혹은 수동으로 Task.cancel()을 이용하여 취소 동작을 전파할 수 있습니다.


---------

### Actors
task들을 격리되어 독립적인 부분으로 프로그램을 분해시키고 이들은 동시에 수행되어도 안전해야합니다. 허나 task 간의 정보 교환이 필요할때가 있습니다. -> 이와같은 경우에서 actor는 concurrent code들 간의 정보 공유를 안전하게할 수 있도록 도와줍니다.

Actor는 래퍼런스 타입입니다. 하지만 일반 클래스와는 다르게 이들의 mutatble한 상태 접근을 한번에 한 task 만이 가능하도록 제한합니다.
```swift
actor TemperatureLogger {
	let label; String
	var measurements: [Int]
	private(set) var max: Int

	init(label: String, measurement: Int) {
		self.label = label
		self.measurements = [measurement]
		self.max = measurement
	} 
}

```
actor의 property나 method에 접근하려고 할때는 await 키워드를 붙여 suspension 포인트임을 명시해야합니다.
```swift
let logger = TemperatureLogger(label: "Some", measurement: 25)
print(await logger.max)	// "25"
```

```swift
extension TemperatureLogger {
	func update(with measurement: Int)  {
		measurements.append(measurmeent)
		if measurement > max {
			max = measurement
		}
	}
}
```
위의 코드에서 update는 이미 actor에서 진행되기 때문에 내부 프로퍼티들을 max 없이 접근가능합니다. 액터는 자신의 상태에 오직 한번에 한 연산만 접근 가능하고 오직 await로 마킹된 suspension point를 만난 경우에만 interrupt 됩니다.

```swift
print(logger.max)	// error
```
logger.max를 외부에서 await없이 접근하려했기에 에러 -> swift는 액터의 내부에서만 격리된 로컬상태에 접근할수있게하고 이를 `actor isolation` 이라 칭함


### Sendable Types
task나 actor를 이용하여 일부 코드들을 동시에 실행시킬수있고, 이들 내부에는 mutable한 상태를 포함할 수 있습니다. -> 이를 `concurrency domain`이라 부릅니다.
어떤 concurrency domain에서 다른 부분으로 공유될수 있는 데이터 타입을 `sendable` type이라 부릅니다.

Sendable protocol을 따르게 하여 커스텀 타입이 sendable 함을 나타낼 수 있습니다. Sendable protocol에는 코드적인 요구사항은 없습니다. 허나 swift가 강제하는 의미적 요구사항이 있습니다.
- 만약 특정 타입이 value 타입이고, 그의 mutable한 상태 또한 다른 sendable 한 타입으로 만들어 진 경우 - ex) 모든 propertry들이 sendable 타입을 준수하는 struct / associated value 또한 sendable을 만족하는 enum
- mutable한 상태가 없고 이 immutable한 상태들이 sendable을 만족할때 - ex) read-only property만 지닌 struct이나 class
- 안전하게 mutable한 상태에 접근할수있는 타입 - ex @MainActor로 마킹된 class나 / 큐나 스레드를 이용해 serial하게 접근을 제어하는 경우 

```swift
struct TemperatureReading {
	var measurement: Int
}
```
TemperatureReading은 sendable한 property만을 가지기에 이는 암묵적으로 sendable입니다.(public이 아니거나 @usableFromInline으로 마킹되지 않은 경우)
-> public이면 왜 안됨?: 아마도 타입이 정의된 타켓 외부에서 sendable이 아님을 extension에 정의할수있어서?

```swift
@available(*, unavailable)
extension TemperatureReading: Sendable { }
```

### Sendable
https://developer.apple.com/documentation/swift/sendable
concurrency 도메인을 걸쳐 그들의 값이 복사를 통해 안전하게 전달될 수 있는 타입

#### Overview
다음의 타입들은 sendable로 표시될수있습니다.
- value type
- mutable한 storage가 없는 reference type
- 내부적으로 상태접근이 관리가 되는 reference type
- @Senable로 마킹된 함수나 클로져

Sendable protocol은 method나 property와 관련된 프로토콜 요구사항이 없지만, semantic requirement가 존재합니다. Sendable Confirmance는 타입이 선언된 부분과 같은 파일에 선언되어야합니다.

@unchecked Sendable로 컴파일러 체크를 생략하며 Sendable을 만족하게 할 수 있습니다 -> @unchecked Senable은 retroactive하게 적용됨 (같은 파일에 Senable comformance가 없어도됨)

#### Sendable Structures and Enumerations
Sendable protocol을 만족하기 위해서는 sendable member나 associated value만을 소유해야함, 이 경우에 structure, enumeration은 암묵적으로 Sendable을 준수함
- Frozen structures and enumerations
- Structures and enumerations that aren’t public and aren’t marked @usableFromInline.

#### Sendable Actors
모든 actor는 암묵적으로 Sendable을 준수한다 => 그들의 mutable한 상태 접근이 순차적으로 보장되기에

#### Sendaable Classes
class가 Sendable을 따르기 위해서는 아래의 조건을 갖춰야한다.
- Be marked final
- Contains only stored properties that are immutable and sendable
- Have no superclass or have NSObject as the superclass

@MainActor로 마킹된 클래스는 암묵적으로 sendable 합니다. 이는 main actor가 내부 상태를 접근하는것을 조율합니다. 이러기에 이 class들은 mutable하거나 nonsendable한 프로포티가 저장될수있습니다.

마찬가지로 @unehecked Sendable을 이용해 컴파일 체크를 피할 수 있음

#### Sendable Functions and Closures
Sendable protocol을 준수하는것 대신에 @Sendable attribute를 사용하여 함수나 클로저를 마킹할 수 있습니다. 이경우에 캡처되는 모든 값은 Sendable 해야하고, sendable closure의 경우에는 sendable한 값의 by-value capture만을 사용해야합니다. 

sendable closure의 겨우에는 요구사항이 만족되면 암묵적으로 sendable 합니다 - ex) Task.detached(priority:operation:)

명시적으로 타입 선언시에 @Sendable annotation을 붙이거나, closure의 파라미터 이전에 @Sendable을 붙여 closure가 Sendable임을 나타낼 수 있습니다.
```swift
let sendableClosure = { @Sendalbe (number: Int) -> String in 
	....
}
```

#### Sendable Tuples
- sendable을 만족하기위해 투플의 모든 엘리먼트들은 다 sendable 해야하고, 만족하는 경우 암묵적으로 Sendable 합니다.

#### Sendable Metatypes
Int.Type과 같은 메타 타입들은 암묵적으로 Sendable protocol을 만족합니다.







