# 왜 우리는 항상 Mutex부터 떠올릴까

비동기 처리나 멀티스레드 환경을 이야기할 때, 대부분 가장 먼저 떠올리는 방법은 `Mutex` 나 `Semaphore` 같은 lock 기반 방식이다.

나 역시 C++ 멀티프로세스 환경에서 shared memory를 다루며 semaphore를 사용했던 경험이 많다.

특정 시점에 하나의 thread만 접근 가능하도록 만들어 상태를 보호하는 방식은 직관적이고 강력하다.

하지만 동시성 문제를 해결하는 방법이 꼭 lock만 있는 것은 아니다.

---

# Lock 기반 방식의 특징

대표적인 lock 기반 방식은 아래와 같은 흐름으로 동작한다.

- lock 획득
- shared state 접근
- 작업 수행
- lock 해제

이 방식은 shared mutable state를 안전하게 보호할 수 있다는 장점이 있다.

하지만 동시에:

- contention
- context switching
- deadlock 가능성

같은 비용도 함께 가져온다.

그래서 동시성 환경에서는 lock을 최소화하거나, 가능하다면 lock 없이 상태를 안전하게 변경하는 방법도 자주 고민하게 된다.

---

# Atomic Operation

CPU는 일부 연산에 대해 atomic instruction을 제공한다.

대표적으로:

- atomic increment
- compare-and-swap(CAS)

같은 연산이 있다.

여기서 atomic이란:

> 연산 도중 다른 thread가 끼어들 수 없는 상태

를 의미한다.

즉:

```text
read
→ modify
→ write
```

과정 전체가 하나의 연산처럼 처리된다.

---

# 간단한 예제

먼저 일반 int를 사용한 코드다.

두 thread가 동시에 counter를 증가시키는 상황을 만들어보자.

```cpp
#include <iostream>
#include <thread>
#include <chrono>

int counter = 0;

void increment(const std::string& name) {
    for (int i = 0; i < 5; i++) {

        int current = counter;

        std::cout
            << name
            << " read: "
            << current
            << std::endl;

        std::this_thread::sleep_for(std::chrono::milliseconds(100));

        counter = current + 1;

        std::cout
            << name
            << " write: "
            << counter
            << std::endl;
    }
}

int main() {
    std::thread t1(increment, "Thread A");
    std::thread t2(increment, "Thread B");

    t1.join();
    t2.join();

    std::cout
        << "Final counter: "
        << counter
        << std::endl;
}
```

---

# 실행 결과

아래와 같은 결과가 발생할 수 있다.

```text
Thread A read: 0
Thread B read: 0
Thread A write: Thread B write: 1
Thread A read: 1
1
Thread B read: 1
Thread B write: 2
Thread B read: 2
Thread A write: 2
Thread A read: 2
Thread B write: 3
Thread B read: 3
Thread A write: 3
Thread A read: 3
Thread B write: 4
Thread B read: 4
Thread A write: 4
Thread A read: 4
Thread B write: 5
Thread A write: 5
Final counter: 5
```

> 증가 연산은 두 번 수행되었지만 실제 값은 한 번만 증가했다.

대표적인 lost update 문제다.

출력 로그가 중간에 섞여 보이는 이유는 `std::cout` 역시 여러 thread에서 동시에 접근하고 있기 때문이다.

이 예제에서는 counter의 lost update를 보여주기 위해 의도적으로 별도의 출력 동기화는 하지 않았다.

---

# Atomic으로 변경

이제 counter를 atomic으로 변경해보자.

```cpp
#include <iostream>
#include <thread>
#include <atomic>

std::atomic<int> counter{0};

void increment() {
    for (int i = 0; i < 100000; i++) {
        counter++;
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);

    t1.join();
    t2.join();

    std::cout << "counter = " << counter << std::endl;
}
```

---

# Atomic 실행 결과

`std::atomic<int>` 를 사용한 경우에는 결과가 안정적으로 유지된다.

```text
counter = 200000
```

여러 thread가 동시에 접근하더라도 상태 유실 없이 정상적으로 증가한다.

---

# 중요한 차이점

여기서 중요한 점은:

> mutex를 직접 사용하지 않았다는 것

이다.

`std::atomic` 은 CPU atomic instruction 기반으로 동작하며, 연산 자체를 atomic하게 수행함으로써 상태 유실을 방어한다.

즉:

> 동시성 문제를 해결하는 방법은 반드시 lock만 있는 것이 아니다.

---

# 마무리

물론 atomic operation이 모든 문제를 해결하는 것은 아니다.

복잡한 상태 변경이나 여러 변수의 일관성을 보장해야 하는 경우에는 여전히 mutex 같은 lock 기반 방식이 필요하다.

하지만:

> 동시성 문제 = 무조건 lock

이라는 관점에서 벗어나기 시작하면, 상태 관리와 성능을 바라보는 시야도 조금씩 달라지기 시작한다.

동시성 문제를 해결하는 방법은 하나가 아니다.

중요한 것은 어떤 문제를 해결하려는지, 그리고 어떤 비용을 감수하고 있는지를 이해하는 것이다.