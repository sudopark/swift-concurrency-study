## Structured Concurrency

: https://www.wwdcnotes.com/notes/wwdc21/10134/

structured concurrency는  static scope를 이용합니다. 이는 코드와 흐름에 대해 쉽게 이해할 수 있도록 돕습니다. 이로인해 코드를 위에서 아래로 읽으며 쉽게 파악할 수 있습니다.

비동기 코드는 값을 리턴하지 않습니다. 함수의 마지막 부분에 값을 반환할 준비가 되어있지 않기 때문입니다. 이는 함수가 나중에 클로저를 통하여 결과를 전달할것임을 의미합니다. 이것은 또한 에러핸들링을 하지 않음을 뜻합니다(에러를 throws하지 않기때문에). 그리고 다른 비동기 콜이 필요하다면 코드블럭이 중첩됩니다.

Aync 함수는 컴플리션 핸들러가 필요 없습니다. 그 대신에 `async`로 마킹되어 값을 리턴합니다. 함수 호출시에 `await`를 이용하여 async 함수의 결과를 사용하기 위해 중첩구문을 사용할 필요가 없습니다. 그리고 또한 컴플리션 핸들러 대신 error를 throws 할수도 있습니다. 이는 더 구조적인 프로그래밍 방법입니다.



Task는 swift의 새로운 기능 입니다. task는 비동기 코드를 작성할 수 있는 실행 context를 제공합니다. 모든 task는 다른 task들과 동시에 돌아갑니다. 가능할때 병렬로 수행됩니다.  또한 컴파일러는 버그를 방지하는것을 돕습니다.

-----

### Async let task

`async let `은 task의 간단한 종류 중 하나입니다.

`let thing = somthing()` 이라는 구문을 쓸때, `something()`이 먼저 계산되고난뒤에 결과가` let thing`에 할당됩니다. 만일 `something()`이 `async` 이고, 먼저 이를 실행하고 이후에 `let thing`에 값을 할당하고자 한다면 `let thing`에 `async`를 붙이세요

```swift
async let thing = something()
```

이를 연산하는 과정에서 child task가 생성됩니다. 이 경우에 `let thing`은 placeholder로 할당됩니다. parent task는 우리가 `thing` 값을 사용하려고 하는 순간까지 실행이 지속됩니다. 이 지점에 `await`를 표시해야 합니다.

```swift
async let thing = something()
// some stuff
makeUseOf(await thing)
```

이때  parent task는 일시중지되고 child task가 완료되어 placeholder를 채울때까지 대기합니다. 만일 async function이 throw 할 수 있다면, `await` 앞에 `try`를 붙이세요.

하나의 스코프에서 여러 aync function을 호출하는 코드는 다음과 같이 쓸 수 있습니다.

```swift
func performAsyncJob() async throws -> Output {
  let (data, _) = try await fetchData()
  let (meta, _) = try await meta()
  return Output(data, meta)
}
```

위의 구문은 처음 `fetchData` 결과를 기다리고 이후에 `meta`를 실행합니다. 그리고 `meta`도 완료된 이후에 `Output`을 리턴합니다.

만일 두 `await` 구문이 서로 독립적이라면, `async let`을 이용하여 이들을 동시에 실행시킬 수 있습니다.

```swift
func performAsyncJob() async throws -> Output {
  async let (data, _) = fetchData()
  async let (meta, _) = meta()
  return Output(try await data, try await meta)
}
```

await를 만나기 전까지 parent task는 중지되지 않고 두 task가 동시에 실행됩니다.

parent task는 한개 이상의 task를 만들 수 있습니다. parent task는 이의 child task들이 완료된 경우에 완료됩니다. 만일 이들 중 하나의 task가 error를 발생시킨다면 parent task는 즉시 종료(exit)됩니다. 이경우에 만일 다른 child task들이 실행중이였다면 parent task는 종료 이전에 이들이 취소되었음을 마킹합니다. 취소되었음을 마킹하는 것만으로는 작업을 취소시킬 수 없습니다. 이는 단지 task와 이 결과가 더이상 필요 없음을 나타냅니다. task는 취소를 핸들링 해야만합니다.  child task가 있는 task가 취소되는 경우에도 child task들은 취소되었다 마킹됩니다.

swift의 task 취소는 협동적(cooperative)입니다. tssk는 취소되었을때 종료되지 않습니다. task는 적절한 지점에서 자신이 취소되었는지 여부를 처리해야합니다 (실제 비동기인지 여부와 상관없이). 특히 실행기간이 긴 작업같은경우에 이를 주의해야합니다. task가 취소되었을때 가능한한 제일 빨리 중지할 수 있도록 하세요.

이는 `try task.checkCancellation( )` 을 호출해 할 수 있습니다. 이는 현재 task가 취소되었는지 확인하고 그럴경우 error를 throw 합니다. `Task.isCancelled`를 이용하는 방법도 있습니다. task가 취소되었을때 task.checkCancellation( ) 처럼 에러를 throw하거나 빈 결과 혹은 일부결과만 반환하게 할 수 있습니다. 호출자가 이를 파악할 수 있도록 하는거에 신경쓰세요

------

### Group task

```swift
func fetchServeralThings(for ids: [String]) async throws -> [String: Output] {
  var output = [String: Output]()
  for id in ids {
    output[id] = try await performAsyncJob()
  }
  return output
}

func performAsyncJob() async -> Output {
  async let (data, _) = fetchData()
  async let (meta, _) = meta()
  return Output(try await data, try await meta)
}
```

위의 코드에서 하나의 id에 대해 task와 두개의 child task가 만들어 집니다. `await performAsyncJob`은 parent이고 `fetchData`, `meta`는 child task를 만들어 냅니다. for loop안에서 `performAsyncJob` 을 `await`라기 때문에 오직 하나의 task만(parent) 활성화될 수 있습니다.

여러개의 `performAsyncJob`을 활성화 시키기위해서 task group을 이용할 수 있습니다. group내에서 만들어지는 task는 group의 scope내에 한정되어야합니다(cannot escape)

`withThrowingTaskGroup(of: Type.self)` 함수를 이용해 task group을 만들 수 있습니다.이 함수는 group 객체를 받는 클로저를 인자로 받습니다. 이 group에 새로운 task들을 추가할 수 있습니다.

```swift
func fetchServeralThings(for ids: [String]) async throws -> [String: String] {
  var output = [String: Output]()
  try await withThrowingTaskGroup(of Void.self) { group in
    for id in ids {
      group.async {
        output[id] = try await performAsyncJob()
      }
    }
  }
  return output
}
// 장풍보소;;
```

child task는 group에 추가될때 바로 시작됩니다.

closure의 마지막 부분에서 group은 scope를 벗어나 추가된 child task들이 대기되기 시작합니다. 이는 `fetchServeralThings` 내에 하나의 task만 있음을 뜻합니다.그리고 이 task는 각각의 `id`로 인해 만들어진 task를 child task로 지니고, 이 child task들도 각자 하위 task들을 지닙니다.

위의 코드는 컴파일 에러가 발생합니다. `output` 변수가 레이스 컨디션의 주된 타겟이 되기 때문입니다. 이는 동시에 진행되는 여러 task들에 의해 수정될 수 있습니다. concurrent code에서 레이스 컨디션은 빈번한 문제입니다. 딕셔너리 타입 자체는 동시성이 보장된 타입이 아닙니다.

task의 생성자는 `@sendable` closure를 주입받고 이는 mutable한 값을 캡처할 수 없습니다. sendable closure는 오직 value 타입이나, actor 그리고 동기화(synchronization)가 구현된 class만들 캡처할 수 있습니다.

이를 고치기 위해 child task들이 dictionary를 수정하는것보다 값을 반환하게 하여 해결할 수 있습니다.

```swift
func fetchServeralThings(for ids: [String]) async throws -> [String: Int] {
    var output = [String: Int]()
    try await withThrowingTaskGroup(of: (String, Int).self) { group in
        for id in ids {
            group.addTask {
                return (id, try await performAsyncJob())
            }
        }
        
        for try await (id, result) in group {
            output[id] = result
        }
    }
    
    return output
}
```

`for try await loop`는 순차적으로 돌아가기때문에 `output` 딕셔너리는 한번에 한번씩 수정됩니다.

task group이 structured concurency를 구성하는 동안 task tree는 살짝 다르게 동작합니다. 만일 그룹내 하나의 task가 실패한다면(throw error) group은 다른 child task들을 모두 취소 시킵니다. 이는 async let의 경우와 동일합니다. 제일 다른점은 group이 scope를 벗어나게 된다면 이 child task들은 취소할 수 없어집니다. task들을 추가하기 위한 클로저에서 빠져나가기전에 `cancelAll`을 호출할 수 있습니다.

-----

### Unstructured tasks

Structured concurrency 명확한 계층구조와 룰이 정해져 있습니다. 하지만 때로는 Unstructured  concurrency가 필요한 경우도 있습니다. 

예를들어 아직 async context가 아닌 부분에서 async 작업을 시작하고 싶을수 있습니다. 이는 아직 task가 없음을 의미합니다. 다른 경우에는 delegate 패턴과 같이 작업이 single scope를 벗어나야 하는 경우도 있습니다.

#### Async tasks

collection view의 delegate에서 비동기 호출을 하고싶은 상황을 가정해보세요.

```swift
// shortened
func cellForRowAt() {
  let ids = getIDs(for: item)  // item is passed to cellForRowAt
  let content = await getContent(for: ids)
  cell.content = content
}

// shortened
func willDisplayCellForItem() {
  let ids = getIDs(for: item)  // item is passed to willDisplayCellForItem
  async {
    let content = await getContent(for: ids)
    cell.content = content
  }
}
```

async 함수는 코드를 현재 액터에서 비동기적으로 실행시킵니다. 이것을 메인 액터로 지정하기위해 class에 @MainActor annotation을 추가하세요;;

```swift
@MainActor
class CollectionDelegate {
  // code
}
```

- unstructured task는 본래의 context에서 액터의 격리성과 우선순위를 이어받습니다.
- 라이프타임은 scope로 제한되지 않습니다.
- 어느지점에서나 시작될수있고
- 반드시 수동적으로 취소하거나 대기해야합니다.

**SIDENOTE** all async work is done in a task. Always.

unstructured task을 이용할때 취소나 에러는 자동적으로 전파되지 않습니다. 이들을 관리하기위해  task들을 다음과같이 보관할 수 있습니다.

```swift
@MainActor
class CollectionDelegate {
  var tasks = [IndexPath: Task.Handle<Void, Never>]()
  
	func willDisplayCellForItem() {
    let ids = getIds(for: item) // item is passed to willDisplayCellForItem
    tasks[item] = async {
      defer { tasks[item] = nil }

      let content = await getContent(for: ids)
        cell.content = content
    }
  }
}
```

task를 보관해서 나중에 이들을 취소할수 있습니다. 그리고 종료되었을때 삭제해야하는데 이는 Defer 구문에서 하고있습니다. 위의 코드에서 취소동작은 없습니다.

이는 main actor에서 진행되기 때문에 `tasks`들에 레이스 컨디션이 발생하지 않음을 보장할 수 있습니다.

-----

### Detacked tasks

종종 actor의 속성을 이어받지 않고 task를 자체적으로 실행시키고 싶을때가 있습니다. 이는 detached task로 할 수 있습니다. 이들은 async task와 동일하게 작동하지만 생성된 context에서 실행되지 않습니다. 우선순위는 파라미터로 전달할 수 있습니다.

아래의 코드에서 이미지를 캐싱하는 부분은 main actor에서 떼어내 진행할 수 있습니다.

```swift
@MainActor
class CollectionDelegate {
  var tasks = [IndexPath: Task.Handle<Void, Never>]()
  
  func willDisplayCellForItem() {
    let ids = getIDs(for: item)
    task[item] = async {
      defer { tasks[item] = nil }
      let content = await getContent(for: ids)
      
      asyncDetached(priority: .background) {
        writeTocache(content)
      }
      
      cell.conetnt = content
    }
  }
}
```

비교적 덜중요한 캐싱 작업은 별도의 액터에서 낮은 우선순위를 지니며 수행될것입니다. detached task에서 task group을 만들 수 있습니다. 이를 통해 여러 비동기 작업을 동시에 진행할 수 있습니다. 그리고 detached task를 이용해 쉽게 취소할 수 있습니다. 왜나면 detached task는 다른 task들의 parent 이기 때문이죠(우선순위도 child task들에 일괄 적용됨)

```swift
asyncDetached(priority: .background) {
  withTaskGroup(of: Void.self) { group in
    group.async { ... }
    group.async { ... }
    group.async { ... }
  }
}
```













