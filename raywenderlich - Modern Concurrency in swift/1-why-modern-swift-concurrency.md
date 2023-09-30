

# Why Modern Swift Concurrency?

swift 5.5에서 asyc, concurrent code 관련 네이티브 concurrency 모델이 추가됨


### 기존에 GCD를 사용하는 방식의 문제점
- Thread explosion: 동시에 스레드가 많이 만들어지면 이들간 스위칭 비용이 많이든다 -> 퍼포먼스 저하
- 우선순위 역전: 낮은 우선순위의 작업이 큐를 선점하고있으면 뒤에 높은 우선순위의 작업이 위치하고있어도 낮은우선순위의 작업이 먼저 진행된다.
- 실행 구조의 부재: 비동기 작업 블럭들은 독립적으로 관리된다 -> 작업을 취소시키거나 접근을 어렵게 한다. 또한 호출지점에서 결과를 얻어내기 가다롭다.

## concurrency 모델 소개
이 새로운 모델은 언어의 신택스적인 특성과 swift runtime + xcode와 관련이 있다. 이는 스레드에 대한 개념을 추상화하고 다음의 특징들을 포함한다.
1. A cooperative thread pool
2. `async`/`await` syntax
3. Structured concurrency
4. Context-aware code complilation

### 1. A cooperative thread pool
새로운 모델은 가용한 CPU 코어에 해당하는 수 많큼의 최대 스레드를 운용한다. -> 런타임에서 스레드 생성, 해제, 스위칭이 필요 없어진다.
대신 귀하의 코드는 잠시 중지 되었다가, 최대한 빨리 나중에 thread pool에서 가용한 스레드에서 재시작된다.

### 2. async/await syntax
swift의 새로운 `async`/`await` syntax는 컴파일러와. swift runtime에게 코드조작이 일시중지 되었다가 재시작 될 수 있음을 알린다. 런타임이 이를 관리해 주기 때문에 스레드나 코어에 대해 생각할 필요 없다.

추가적으로 새로운 신택스는 강한 참조된 self에 대해 고민할 필요가 없게 만든다 -> 콜백으로 escaping closure를 넘길 필요가 없기 때문에(하지만 주의 필요)

### 3. Structrured concurrency
모든 비동기 작업들은 parent task의 계층에 속해게 되고 작업의 우선순위를 부여받을 수 있다. 이 계층구조를 이용하여 런타임에서 모든 차일드 테스트들은 부모 테스크가 취소되면 같이 취소된다. 더욱이 모든 차일드 테스크가 끝날때까지 부모 테스크를 기다리게 할 수 있다.
이 특징은 task간 우선순위 조정에 매우 유용하다.

### 4. Context-aware code complilation
컴파일러는 지속적으로 주어진 코드가 비동기적으로 실행될 수 있는지 검사한다. 또한 공유된 자원을 수정하는것과 같이 개발자가 잠재적으로 안전하지 않은 코드를 짜는것을 방지한다.
컴파일레벨에서의 체크는 새로운 기능인 actor에서 더 득을 볼 수 있다. actor는 자신의 상태에 접근하는 것을 동기/비동기 적으로 구분하고 데이터가 부패되는것을 컴파일레벨에서 방지한다.


## Writing your first async/await
예시코드
```swift
func availableSymbols() async throws -> [String] {
	guard let url = URL(string: "...")
	else  {
		throw ...
	}
	let (data, response) = try await URLSession.shared.data(from: url)
	guard (response as? HTTPURLResponse)?.statusCode == 200 else {
 		 throw ...
	}
	return try JSONDecoder().decode([String].self, from: data)
}
```
예시 코드에서 `async` 키워드를 이용해 컴파일러에게 코드가 비동기 컨텍스트에서 실행됨을 알려라. 다시말해서 이 코드는 일시중지/재시작 될 수 있음을 뜻한다. 또한 이는 작업이 얼마나 걸리든 값을 반환함을 의미한다(동기 코드처럼)

메소드 내에서 `async`인 `URLSession.data(from:delegate:)`을 호출하면 `availableSymbols()`은 일시 중지하고 데이터조회가 완료되면 다시 시작한다.
`await`를 이용하여 런타임 일시중지 지점을 제공하다. -> 이는 해당 메소드가 일시중지되어있고 다른 task들이 먼저 실행되고 난 이후에 다시 메소드가 재시작할 수 있음을 의미한다.


