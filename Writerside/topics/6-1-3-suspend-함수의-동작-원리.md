# 6.1.3 suspend 함수의 동작 원리

## 1. `suspend` 함수의 컴파일 방식

Kotlin 컴파일러는 `suspend` 키워드가 붙은 함수를 일반 함수와 다른 형태로 변환합니다. 구체적으로는 다음과 같은 과정을 거칩니다.

1. Continuation 인자 추가

```Kotlin
suspend fun someFunc(param: Type): ReturnType {
    ...
} 
// -> 내부적으로 아래와 같이 변환
fun someFun(param: Type, continuation: Continuation<ReturnType>): Any? {
    ...
}
```
위처럼 suspend 함수는 내부적으로 현재 함수의 상태를 담을 수 있는 Continuation 객체를 추가로 주고 받게 됩니다. 

2. 상태 기계(State Machine)코드 생성

`suspend`함수 내에는 일시 중단 지점(`suspend` 키워드로 호출되는 다른 함수나, `yield`, `withContext` 등)이 있을 수 있는데, 이를 **중단점(suspension point)** 이라고 부릅니다.

- 컴파일러는 함수 내의 중단점을 기준으로 상태(label)을 구분하고, 해당 상태 간 전환 로직을 자동으로 만들어줍니다.
- 예를 들어, 첫 번째 중단점 전후로 `label=0`, `label=1` 처럼 레이블이 붙고, 중단점을 지나면 `label=1`에서 로직을 이어가도록 변환합니다.

<deflist collapsible="true">
    <def title="원본 코드" default-state="expanded">
        <code-block lang="kotlin">
            suspend fun greet() {
                println("Hello")
                delay(1000)
                println("World")
            }
        </code-block>
    </def>
    <def title="내부적으로 변환된 코드" default-state="collapsed">
        <code-block lang="kotlin">
            // 내부적으로 컴파일된 가상의 코드 예시
            fun greet(continuation: Continuation&lt;Unit&gt;): Any? {
                // state machine을 위해 label 변수를 사용
                when (continuation.label) {
                    0 -> {
                        // 일단 label을 다음 단계로 변경
                        continuation.label = 1
                        // 첫 번째 구문: println("Hello")
                        println("Hello")
                        // delay(1000) 호출 -> 일시 중단 처리
                        // delay 함수도 suspend 함수이므로, 또다른 continuation을 사용
                        // delay가 끝나면 resume(계속) 지점으로 돌아옴
                        return delay(
                            timeMillis = 1000,
                            // delay가 끝나면 실행할 콜백
                            continuation = object : Continuation&lt;Unit&gt; {
                                override val context = continuation.context
                                override fun resumeWith(result: Result&lt;Unit&gt;) {
                                    // delay가 끝났으니, greet의 원래 continuation을 다시 이어감
                                    greet(continuation) 
                                    // 여기서 greet(...) 재호출 시, continuation.label == 1 상태로 들어가게 됨
                                }
                            }
                        )
                    }
                    1 -> {
                        // 여기서는 delay 이후 "다음 단계"를 진행
                        continuation.label = 2
                        // 두 번째 구문: println("World")
                        println("World")
                        // 함수 끝 -> Unit 반환
                        return Unit
                    }
                    else -> {
                        // 이미 끝난 함수인데 재개가 또 들어온 경우 등
                        throw IllegalStateException("Already completed or invalid state")
                    }
                }
            }
        </code-block>
    </def>
</deflist>

3. 재개(resume) 로직 삽입

실행이 중단된 함수는, 이후에 "재개(resume)" 메서드를 호출해 이어갈 수 있어야 합니다.
그래서 Continuation 객체에는 `resumeWith(result: Result<T>`과 같은 콜백 함수가 존재합니다. 

- 비동기 작업(예: 네트워크)이 끝나면, 해당 콜백을 통해 "이제 함수를 이어서 실행해도 좋다"는 신호와 함께 결과를 전달합니다.
- 이를 통해, 코루틴은 "원래 실행 중이던 함수"의 특정 지점부터 다시 로직을 이어갈 수 있게 됩니다.

즉, `suspend`함수는 단순히 "중단과 재개가 가능한" 기계로 컴파일되어, 내부 상태와 실행 위치(레이블)를 Continuation 객체에 저장해놓습니다.

## 2. Continuation 객체의 역할

`suspend` 함수가 중단되었다가 재개될 때, 어디서부터 다시 시작해야 하는지, 지역 변수 값은 얼마였는지 등의 정보를 기억하는 것이 중요합니다. 
이 역할을 수행하는 것이 Continuation 객체입니다. 

- 상태 필드: 현재 레이블(현재 진행 중인 위치), 함수 내부 지역 변수들, 예외 처리 흐름 등이 저장됩니다. 
- resume 및 resumeWithException: 비동기 처리 완료 시, 성공 결과 또는 에러를 전달하며 함수를 재개할 수 있습니다. 
- 코루틴 컨텍스트(Coroutine Context): 디스패처(Dispatchers.IO, Dispatchers.Main 등), Job, 예외 핸들러 등의 정보를 담습니다. 이 정보들도 Continuation을 통해 전달, 관리됩니다.

## 3. 비동기 처리와의 결합

`suspend` 함수는 코루틴이 "OS 스레드를 직접 막지 않고도" 다른 작업을 수행할 수 있도록 해줍니다.

- `delay`, `withContext`, `await` 등 내부에서 **중단점**을 만날 때마다, 해당 코루틴은 일시 중단되고, 실행 중인 스레드는 즉시 다른 코루틴으로 넘어갈 수 있게 됩니다. 
- 이후 조건이 충족되거나, 비동기 작업이 완료되면 "다시 이 코루틴을 재개해도 좋다"는 신호와 함께 이전 상태를 복원하여 이어서 실행합니다. 

이런 구조는 OS 레벨의 스레드 전환이 아닌, 사용자 레벨에서 이뤄지므로 오버헤드가 매우 작고, 동시에 많은 코루틴을 관리할 수 있게 됩니다.

> **suspend 함수 호출 자체가 항상 중단점을 의미할까?**
> 
> 모든 `suspend` 함수 호출이 반드시 실제 중단점이 되는 것은 아닙니다.
> `suspend` 함수가 내부적으로 실제로 중단을 일으키는 지점(`delay`, `withContext`, `await` 등)을 전혀 사용하지 않는다면
> 중단 가능성만 있을 뿐 실제로는 중단이 일어나지 않는 함수가 됩니다.
> 
> 실제 중단점이 없는 `suspend` 함수는 일반 함수와 다를 바 없이 동작하므로, context switching(스레드 전환)이 발생하지 않습니다.
> 호출시 점에 중단점이 형성되긴 하지만, 실제 중단이 일어날 가능성은 없습니다.