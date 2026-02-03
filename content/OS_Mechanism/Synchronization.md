
Synchronization (동기화)는 여러 실행 흐름(스레드([[Thread]]), 인터럽트([[Interrupt]]), 커널 작업)이 동시에 같은 데이터를 접근할 때, 데이터가 깨지지 않도록 접근 순서를 강제하는 메커니즘을 의미한다.

동기화는 왜 필요할까? 동기화의 필요성은 간단한 예시로 쉽게 알 수 있다.

멀티스레드에서는 다음과 같은 일이 자연스럽게 발생할 수 있다:
```
// Thread A
if (ptr != NULL)
    use(ptr);

// Thread B
free(ptr);
ptr = NULL;
```

논리적으로 보면 이 코드는 안전해 보인다.

  - Thread A에서는 ptr이 NULL이 아닐 경우 ptr을 사용하고,
  - Thread B는 ptr을 free한 뒤 NULL로 만들어 dangling pointer를 제거한다.

그러나 실제 실행에서는 스케줄링([[Scheduling]])과 컨텍스트 스위치([[Context Switch]])로 인해
아래의 순서가 발생할 수 있다:
```
Thread A: if (ptr != NULL)
--- context switch ---
Thread B: free(ptr)
Thread B: ptr = NULL
--- context switch ---
Thread A: use(ptr)
```

만약 해당 상황에서 ptr에 유효한 주소가 남아있다고 가정해보자.

Thread A는 `ptr`을 검사할 때 이미 `ptr` 값을 레지스터에 로드해 두었고, Context Switch 후에도 그 레지스터 값이 그대로 복원된다.
이때 Thread B가 `ptr = NULL`로 메모리를 수정했더라도, Thread A는 ptr을 다시 읽지 않기 때문에 이미 **free된 주소를 그대로 사용(Use-After-Free)**하게 된다.

이처럼 여러 실행 흐름이 같은 데이터를 동시에 접근하면, 그 실행 순서에 따라 프로그램의 의미 자체가 바뀌어 버린다.

따라서 동기화는 이렇게 동시에 데이터에 접근하는 경우가 생길 때,
공유 상태를 지키기 위해 실행 순서를 커널과 CPU 수준에서 강제하는 메커니즘이다.