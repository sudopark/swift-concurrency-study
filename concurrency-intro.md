## Swift Concurrency

[Concurrency - The Swift Programming Language (Swift 5.5)](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)

: swift는 비동기 + 병렬 코드를 지원합니다. 비동기 코드는 프로그램의 다른 부분이 실행되는 도중에 얼마동안 중지되었다가 재시작 할 수 있습니다.

일시중지 및 재시작은 네트워크에서 파일을 다운받거나 파싱하는 등 오랜 시간이 걸리는 작업 중에 UI를 업데이트 치는 작업을 예로 들 수 있습니다. 또한 병렬 코드는 4개의 코어 프로세서를 지닌 컴퓨터가 동시에 4개의 코드를 실행할 수 있듯 동시여 여러 코드 조각들이 실행될 수 있음을 의미합니다.

비동기 병렬 코드는 프로그램의 복잡도를 증가시킵니다. Swift를 이용하면 이 의도를 컴파일 타임의 체킹으로만도 쉽게 표현할 수 있습니다. 예를들어 가변 상태를 안전하게 다루기 위해 actor를 이용하는것 처럼 말입니다.

하지만 느리고 버그발생이 농후한 코드에 동시성을 추가하는것은 처리속도가 더 빨리지거나 정확함을 보장할수 있다고 장담할수는 없습니다. 사실 동시성을 추가하는것은 디버깅을 더 어렵게 하기도 합니다. 하지만 Swift가 언어 레벨에서 지원하는 동시성 관련 특성들을 위의 문제들을 컴파일 시간에 잡아낼수있도록 돕습니다.



### Defining and Calling Asynchronous Functions

비동기 함수나 메소드는 실행 도중에 일시 정지될 수 있는 특별한 함수입니다. 이는 완료하거나, 에러를 던지거나, 아무 값도 리턴하지 않을때까지 수행되는 일반 동기 함수와는 다릅니다. 비동기 함수나 메소드는 위의 속성을 그대로 지니지만 특별하게도 도중에 중지되고 다시 시작될때까지 기다릴 수 있습니다.

함수나 메소드가 비동기라는것을 나타내기 위해 파라미터 선언 이후에 ```async``` 를 추가하세요. (리턴값이 있는 경우에는 -> 이전에). 이는 에러를 던지는 함수들을 나타낼때 ```throws```를 적시하던것과 비슷합니다. (비동기이고 error를 던지는 케이스라면 ```throws``` 이전에 ```async```를 적으세요)

```swift
func listPhotos(inGallery name: String) async -> [String] {
  let result = // .... some async networking code ...
 	return result
}
```

비동기 함수를 호출할때 함수가 반환될때까지 실행은 중지됩니다. 이와같이 일시중지가 예상되는 함수 호출 지점에 ```await```를 붙여줘야합니다. 이는 throwing function에 대해 호출시 ```try```를 붙이는 것과 동일합니다.(에러 발생시 프로그램 실행 플로우를 바꾸겠다는 의미) 비동기 메소드 내에서 실행 흐름은 다른 비동기 메소드를 호출할때만 일시중지 됩니다. 이 지점은 절대 암시적이거나 선점적이지 않습니다. 즉 가능한 모든 중단지점이 ```await```로 표시됩니다.

```swift
let photoNames = await listPhotos(inGallery: "Summer Vacation")
let sortedNames = photoNames.sorted()
let name = sortedNames[0]
let photo = await downloadPhoto(named: name)
show(photo)
```

위의 예시에서 `listPhotos(inGallery:)`, `downloadPhoto(named:)` 함수들은 네트워크 요청이 필요하고 완료되는데 상대적으로 긴 시간이 걸릴수있습니다. 이들을 반환(->) 이전에 `async`로 표시해 비동기 함수로 만듬으로써 이들의 반환을 기다리는 동안 앱의 다른 코드들이 수행될 수 있게 합니다.

#### 실행 순서

concurrency의 특성을 이해하기위해 위의 예시를들어 가능한 실행 순서를 예로 들어보겠습니다.

1. 첫번째 라인부터 코드가 실행되어 천번째 `await` 구문을 만납니다. 그리고 `listPhotos(inGallery:)` 함수를 호출하고 함수가 리턴될때까지 기다립니다.
2. 해당 코드가 일시정지된동안 동일한 프로그램에서 다른 코드가 수행됩니다. 예를들어 오랫동안 백그라운드에서 갤러러의 새로운 사진들을 업데이트 치는 작업을 들 수 있습니다. 이 코드는 `await`로 표시된 다음 일시 중단 지점까지 또는 완료될때까지 실행됩니다.
3. `listPhotos(inGallery:)` 리턴 이후에 이전 중지지점부터 코드가 다시 실행되고 함수의 결과깂을 `photoNames` 변수에 할당합니다.
4. `sortedNames`, `name` 들이 있는 줄의 코드는 일반 동기 코드입니다. 이 줄에는 `await`가 없기 때문에 일시중단 가능한 지점이 존재하지 않습니다.
5. 다음 `await`는 `downloadPhoto(name:)` 함수 호출 지점에 위치하고, 해당 코드는 값이 반환될때까지 중지되어 다른 동시성 코드들이 실행될 수 있는 기회를 제공합니다.
6. `downloadPhoto(name:)` 반환 이후에 변수 `photo`에 할당되고 `show(_:)` 함수 호출시에 인자로 전달됩니다.

귀하의 코드에서 가능한 중지 포인트는 `await` 로 나타내세요. 이는 이 코드가 비동기 함수나 메소드가 반환될때까지 중지될 수 있음을 나타냅니다. 이는 다른말로 스레드 양보(yielding the thread)라 부릅니다. 기저에서 swift는 해당 코드의 실행을 일시중지하고 동일 스레드에서 다른 코드를 실행시키기 때문입니다. await가 마킹된 코드는 일시중지될수 있어야 하기때문에 비동기 함수나 메소드를 호출할 수 있는 프로그램의 특정 지점이여야만 합니다.

- Code in the body of an asynchronous function, method, or property
- Code in the static main ( ) method of a structure, class, or enumeration that's marked with @main
- Code in a detached child task, as shown in [Unstructured Concurrency](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html#ID643) below

-------



### Asynchronous Sequences

위의 예제에서 listPhotos(inGallery:) 함수는 어레이의 엘리먼트를 모두 준비한 이후에 비동기적으로 값을 반환합니다. 이와 다르게 또다른 방식으로 *asynchronous sequence*를 이용해 콜렉션의 각 엘리먼트를 하나씩 기다리는 것 입니다. 비동기 시퀀스를 순회는 아래와 같습니다.

```swift
import Foundation

let handle = FileHandle.standardInput
for try await line in handle.bytes.lines {
  print(line)
}
```

일반적인 for 문과는 다르게 for 이후에 await를 적습니다. 이는 비동기 함수나 메소드를 호출하는 것과 비슷하고 await는 가능한 일시 중단 지점이 있음을 나타냅니다. for-await-in loop는 각 순회의 처음에 엘리먼트가 준비되었을때까지 일시 중지 됩니다.

타입이 Sequence을 따르게하여 for-in loop 구문을 사용할 수 있었던것처럼, [`AsyncSequence`](https://developer.apple.com/documentation/swift/asyncsequence) 프로토콜을 따르게하여 for`-`await`-`in loop를 사용할 수 있게 할 수 있습니다.

-----



### Calling Asynchronous Functions in Parallel

await와 함께 비동기 함수를 호출은 오직 하나의 코드조각만 실행 가능합니다. 비동기 코드가 실행되는동안, 호출자는 다름줄로 넘어가기 이전에 작업이 끝나기를 기다립니다. 아래의 예제는 처음 3개의 사진을 갤러리에서 불러오는 코드입니다.

```swift
let firstPhoto = await downloadPhoto(named: photoNames[0])
let secondPhoto = await downloadPhoto(named: photoNames[1])
let thirdPhoto = await downloadPhoto(named: photoNames[2])

let photos = [ firstPhoto, secondPhoto, thirdPhoto ]
show(photos)
```

위와 같은 방식은 결점이 있습니다. 다운로드 작업 자체는 비동기적이고 실행중에 다른 작업이 진행되는것을 허용하지만, 순차적으로 다운로드가 하나씩 진행되게 짜야였습니다. 위와 같은 방식은 위의 예제에서 적절하지 않습니다. 각각의 사진들은 독럽적으로 다운로드 될 수 있고 또한 모두 동시에 진행될 수 있습니다.

비동기 코드를 주변의 코드들과 함께 병렬적으로 실행시키려면 상수를 선언하는 let 앞에 async를 명시하세요. 그리고 이 상수 값들을 이용할때 await를 적으세요

```swift
async let firstPhoto = downloadPhoto(named: photoNames[0])
async let secondPhoto = downloadPhoto(named: photoNames[1])
async let thirdPhoto = downloadPhoto(named: photoNames[2])

let photos = await [ firstPhoto, secondPhoto, thirdPhoto ]
show(photos)
```

위의 코드에서 3가지 사진 다운로드 작업들은 앞선 다운로드 작업의 종료 여부와 상관없이 시작될 수 있습니다. 그리고 시스템 리소스가 가능하다면 동시에 진행될 수도 있습니다. 이 함수들의 호출에는 결과값 반환을 기다리기위해 일시정지할 필요가 없기 때문에 await가 쓰이지 않습니다. 대신 photos 선언부까지 실행됩니다. - 이 지점에서 프로그램은 3개의 다운로드 완료된 사진이 필요하기 때문에 await를 적어야 합니다.

위의 서로 다른 두가지 방식에 대해 다음과 같이 생각해볼 수 있습니다.

- 후행 작업에서 함수 호출 결과를 필요로 하다면 await로 호출하세요. 이 경우에 작업은 순차적으로 진행됩니다.
- 후행 작업에서 함수 호출 결과를 기다릴 필요가 없다면 async-let과 함께 함수를 호출하세요. 이 경우에 작업은 병렬적으로 진행됩니다.
- await, async-let 모두 이들이 중지되었을때 다른 코드들이 실행되는것을 허용합니다.
- 두가지 경우 모두 일시 중지가 예상되는 지점에 await를 명시해야합니다.

당연히 이 두가지 방식을 같이 사용하여 코드를 작성 할 수 있습니다.

----



### Tasks and Task Group

task는 프로그램에서 비동기로 수행되는 작업의 단위입니다. 모든 비동기 코드들은 특정 task의 부분으로 동작합니다. 이전 섹션에서 이야기했던 async-let 구문은 child task를 만들어 냅니다. 또한 task group을 만들 수 있고 여기에 child task를 추가할 수 있습니다. 이를통하여 우선순위를 조절하거나 취소작업을 할수있고, 동적으로 여러 task를 만들수 있게합니다.

task들은 계층구조로 정렬됩니다. task group 내 각각의 task는 동일한 parent task를 지니고, child task를 지닐수도 있습니다. task와 task group간 명시적인 관계에로 인해 이와같은 방식을 structured concurrency라 부릅니다. 비록 정합성에 대해 코드를 짜는 사람이 신경을 써야하기는 하지만, task간 명시적인 parent-child 관계로 인해 swift는 취소와 같은 동작을 핸들링하거나 컴파일 타임에서 에러를 감지해낼 수 있습니다.

```swift
await withTaskGroup(of: Data.self) { taskGroup in
  let photoName = await listPhotos(inGallery: "Summer Vacation")
  for name in photoNames {
    taskGroup.addTask { await download(named: name)}
  }
}
```

더 자세한 내용은 [TaskGroup](https://developer.apple.com/documentation/swift/taskgroup)을 참조하세요

#### Task Cancellation

swift concurrency는 협동적인 취소 모델을 사용합니다. 각각의 task들은 적절한 실행지점에서 이미 취소되었는지를 검사하고 적절한(?) 방법으로 취소에 응답합니다. 수행중이던 작업에 따라 이는 대게 아래의 경우와 같습니다.

- Throwing an error like CancellationError - CancellationError에러를 스로우
- Returning nil or an empty collection - nil이나 빈 콜렉션을 반환
- Returning the partially completed work - 일부만 완료된 작업을 반환

취소되었는지 확인하기 위해 CancellationError를 스로우 하는 경우에는 [`Task.checkCancellation()`](https://developer.apple.com/documentation/swift/task/3814826-checkcancellation)을 호출해 task가 이미 취소되었는지를 확인해보거나 [`Task.isCancelled`](https://developer.apple.com/documentation/swift/task/3814832-iscancelled) 값을 검사해 취소 여부를 핸들링 하세요. 사진을 다운로드 받는 task 예제에서 취소된경우에는 다운로드중이던 파일을 삭제하거나 네트워크 작업을 취소하는 작업이 필요할 수 있습니다.

취소를 수동으로 전파하려면 [`Task.cancel()`](https://developer.apple.com/documentation/swift/task/3851218-cancel)을 호출하세요

-----



### Actors

클래스와 같이 액터는 참조타입이므로 발류타입과의 비교와의 설명에서 클래스타입이 그랬던것처럼 동일합니다. 클래스와 다르세 actor의 가변적인 상태에서는 한번에 하나의 task만이 접근 가능합니다. 이는 동일한 actor를 다루는 여러 task들 사이에서도 안전성을 보장합니다. 아래는 기온을 기록하는 엑터의 예시 입니다.

```swift
actor TemperatureLogger {
  let label: String
  let measurements: [Int]
  private(set) var max: Int
  
  init(label: String, measurement: Int) {
    self.label = label
    self.measurements = [measurement]
    self.max = measurement
  }
}
```

actor 키워드 이후에 중괄로 엑터를 정의합니다. TemperatureLogger 엑터는 외부에서 접근 가능한 프로퍼티를 지니고 max 프로퍼티는 오직 엑터 내부에서만 수정되도록 제한합니다.

structure나 class의 생성자와 경우와 동일한 방식으로 actor 인스턴스를 만들 수 있습니다. actor의 프로퍼티나 함수에 접근하려할때 await를 이용하여 잠정적인 일시중단 지점임을 나타내야합니다.

```swift
let logger = TemperatureLogger(label: "Outdoors", measurement: 25)
print(await logger.max)
// Prints "25"
```

위의 예제에서 max값을 어세스하는것은 잠정적인 일시중단 지점입니다. 이는 actor가 한번에 오직 하나의 task만 가변 상태에 접근하는것을 허용하기 때문입니다. 만일 다른 코드가 logger와 상호작용하고있다면 이 코드는 프로퍼티에 접근하기 위해 기다려지게됩니다.

반대로  actor 내부에서 프로퍼티를 다룰때는 await가 필요없습니다. 

```swift
extension TemperatureLogger {
  func update(with measurement: Int) {
    self.measurements.append(measurement)
    if measurement > max { max = measurement }
  }
}
```

위의 예제에서 `update(with: )` 메소드는 이미 actor 내부에서 수행되기 때문에 max를 외부에서 읽을때 await를 사용하였던것과 같은 작업이 필요없습니다. 이 메소드는 왜 액터가 필요한지 나타내는데 액터의 상태 업데이트는 일시적으로 불변성을 초래할 수 있기 때문입니다. (대충 내부에 크리티컬 섹션이 존재한다는 말 생략) 여러 테스크가 동일한 인스턴스와 동시에 상호작용하는것을 방지하면 다음과 같은 상황에서 발생하는 문제를 방지할 수 있습니다.

1. update(with:) 메소드를 호출하면 제일먼저 measurements 어레이를 업데이트합니다.
2. 다음으로 max를 업데이트하기 이전에 코드의 다른 부분에서 max와 measurements 값을 읽으려합니다.
3. max를 업데이트하고 코드가 끝납니다.

위와같이 update 메소드가 다 끝나기 이전에 로거의 프로퍼티를 읽으려한다면 값이 일시적으로 올바르지 않을 수 있습니다. 이를 액터를 이용하여 그들의 상태를 다루는데 하나의 작업만을 허용하는 액터를 이용하여 방지할 수 있습니다. udpate(with:) 는 일시중단 지점을 포함하지 않음으로 update 도중에 어떤 코드도 이에 접근할 수 없습니다.

클래스의 인스턴스의 프로퍼티를 읽는것처럼 액터 외부에서 이들을 접근하려하면 다음과 같이 컴파일 에러가 발생합니다.

```swift
print(logger.max)  // Error 
```

actor의 프로퍼티는 actor의 격리된 로컬 상태이므로 await 없이 접근하는것은 실패합니다. 이를 통해 swift는 오직 액터 내부에서만 액터의 로컬 상태를 접근할 수 있음을 보장합니다. 이는 actor isolation이라 알려져 있습니다.
