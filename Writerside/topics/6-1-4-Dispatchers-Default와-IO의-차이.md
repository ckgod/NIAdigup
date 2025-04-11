# 6.1.4 Dispatchers.Default와 IO의 차이

코루틴에서 Dispatcher는 코루틴이 실행될 스레드, 스레드 풀을 결정하는 역할을 하는 CoroutineContext 중 하나이다.
`Dispatchers.IO`와 `Dispatchers.Default`는 같은 스레드 풀을 공유하는 것으로 잘 알려져 있다. 그러면 정확히 어떤 차이가 있을까?

간단히 결론만 요약하자면,
- **`Dispatchers.Default`** 는 CPU 집약적인 작업에 최적화되어, 코어 수 이하(또는 코어 수에 제한된)의 스레드를 사용하려고 한다.
- **`Dispatchers.IO`** 는 블로킹 I/O 작업에 최적화되어, 필요 시에는 더 많은 스레드를 늘려서 사용할 수 있도록 설계되었다.

`Dispatchers.IO`가 더 많은 스레드를 사용하는 것은 아래 코드로 테스트해볼 수 있다.

```Kotlin
fun main() = runBlocking {
    val ioThreads = mutableSetOf<String>()
    val defaultThreads = mutableSetOf<String>()

    repeat(50) {
        launch(Dispatchers.IO) {
            ioThreads.add(Thread.currentThread().name)
            delay(1)
        }
    }
    repeat(50) {
        launch(Dispatchers.Default) {
            defaultThreads.add(Thread.currentThread().name)
            delay(1)
        }
    }

    delay(150)

    println("IO: ${ioThreads.size}")
    println("Default: ${defaultThreads.size}")
    println("IO names: $ioThreads")
    println("Default names: $defaultThreads")
}
```

출력 결과
```console
IO: 36
Default: 12

IO names: [DefaultDispatcher-worker-1, DefaultDispatcher-worker-3, 
            DefaultDispatcher-worker-2, DefaultDispatcher-worker-6,
            DefaultDispatcher-worker-5, DefaultDispatcher-worker-8,
            DefaultDispatcher-worker-4, DefaultDispatcher-worker-11,
            DefaultDispatcher-worker-10, DefaultDispatcher-worker-7,
            DefaultDispatcher-worker-13, DefaultDispatcher-worker-15,
            DefaultDispatcher-worker-9, DefaultDispatcher-worker-17,
            DefaultDispatcher-worker-12, DefaultDispatcher-worker-16,
            DefaultDispatcher-worker-19, DefaultDispatcher-worker-20,
            DefaultDispatcher-worker-21, DefaultDispatcher-worker-23,
            DefaultDispatcher-worker-18, DefaultDispatcher-worker-14,
            DefaultDispatcher-worker-25, DefaultDispatcher-worker-24,
            DefaultDispatcher-worker-27, DefaultDispatcher-worker-26,
            DefaultDispatcher-worker-22, DefaultDispatcher-worker-32,
            DefaultDispatcher-worker-30, DefaultDispatcher-worker-29,
            DefaultDispatcher-worker-33, DefaultDispatcher-worker-35,
            DefaultDispatcher-worker-28, DefaultDispatcher-worker-34,
            DefaultDispatcher-worker-31, DefaultDispatcher-worker-36]
Default names: [DefaultDispatcher-worker-16, DefaultDispatcher-worker-33,
                DefaultDispatcher-worker-30, DefaultDispatcher-worker-27,
                DefaultDispatcher-worker-36, DefaultDispatcher-worker-3,
                DefaultDispatcher-worker-10, DefaultDispatcher-worker-11,
                DefaultDispatcher-worker-24, DefaultDispatcher-worker-28,
                DefaultDispatcher-worker-9, DefaultDispatcher-worker-42]
```

위 결과처럼 IO 디스패처를 사용하면 코루틴이 더 많은 스레드에서 동작할 수 있다. 

조금 더 구체적으로 알아보자.

### 1. `Dispatchers.Default`

- 역할
  - CPU 바운드 작업에 최적화된 디스패처
  - 예: 대규모 연산, CPU를 많이 사용하는 알고리즘 실행 등
- 스레드 풀 크기
  - 코어 수를 기반으로 스레드 풀 크기를 제한한다. (예: 4코어 환경에서는 4~8개 정도)
  - 이는 CPU 사용률을 최대한 높이되, 과도한 Context Switching 이나 불필요한 스레드 생성을 막기 위한 설정이다.

- 왜 이렇게 설계 되었을까?
  - CPU가 처리할 수 있는 "병렬 연산" 양은 어느 정도 물리적 한계가 있다.
  - 단순히 스레드를 늘린다고 해서 CPU 연산 속도가 무한정 빨라지지 않는다.
  - 따라서 CPU 코어 수만큼(혹은 조금 상회하는 정도)의 스레드를 사용하는 것이 성능 면에서 가장 효율적이다. 

### 2. `Dispatchers.IO`

- 역할
  - 블로킹 I/O(파일, 네트워크 통신 등)을 수행하는 작업을 처리하기 위한 디스패처
  - 예: 파일 입출력, DB 쿼리, 네트워크 요청 등
- 스레드 풀 크기
  - 내부적으로는 `Dispatchers.Default` 와 동일한 ForkJoinPool을 사용하지만, I/O가 많은 작업에서 오랫동안 블로킹이 발생해도 시스템의 처리량을 유지하기 위해 스레드를 더 많이 생성할 수 있게끔 헝요된다.
  - 결과적으로 I/O 작업이 많을 때. 필요한 만큼 스레드가 증가하여 "놀고 있는 CPU가 있지 않도록" 한다.

- 왜 이렇게 설계 되었을까?
  - I/O 작업은 스레드가 대기 시간(블록되는 시간)이 길어질 수 있다.
    - 예를 들어, 파일이나 네트워크 응답을 기다리는 동안 해당 스레드는 아무 일도 하지 못하고 쉬게 된다. 이때, "CPU는 놀고 있는데, 스레드는 대기 중인" 상태가 많아지면 전체 처리량이 낮아진다. 이 문제를 해결하기 위해, `Dispatchers.IO` 는 CPU 코어 수보다 더 많은 스레드를 확보할 수 있도록 설계되었다.
  - 대기 시간이 긴 I/O 작업이 많아도, 필요한 만큼 추가 스레드가 투입되어 CPU를 사용할 수 있게 된다.

참고: [CPU, 코어, 스레드, 스레드 풀(Thread Pool)](6-e-CPU-코어-스레드-스레드-풀.md)