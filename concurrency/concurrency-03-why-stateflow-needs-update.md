# 왜 StateFlow의 update{} 가 필요할까

> 개별 읽기와 쓰기가 안전해도, 읽고 계산하고 쓰는 전체 과정이 원자적이라는 보장은 없다.

처음 Kotlin의 `StateFlow` 를 접했을 때는 꽤 인상적이었다.

특히:

```kotlin
_state.value = ...
```

형태로 상태를 관리하는 방식은 기존 mutable state를 직접 수정하는 방식보다 훨씬 안전해 보였다.

그리고 내부적으로 atomic하게 동작한다는 이야기를 접하면서:

> "아, 이제 동시성 문제도 대부분 안전하게 방어되는 건가?"

라고 생각했었다.

---

실제로 내가 작업했던 프로젝트에서도 큰 문제는 없었다.

route request를 처리하는 API thread와 state를 변경하는 thread의 역할을 어느 정도 분리해두었고, 상태 변경 역시 대부분 Main Thread에서만 수행되도록 구성되어 있었기 때문이다.

즉:

- single writer에 가까운 구조였고
- 여러 thread가 동시에 동일 state를 수정하는 상황 자체가 거의 없었다.

그래서 실제로는:

```kotlin
_state.value = _state.value.copy(...)
```

패턴만으로도 충분히 안정적으로 동작했다.

---

하지만 이후 동시성 관점에서 코드를 다시 바라보며 생각이 조금 달라졌다.

예를 들어 아래와 같은 코드를 보자.

```kotlin
_state.value = _state.value.copy(
    count = _state.value.count + 1
)
```

처음 보면 immutable object를 사용하고 있기 때문에 꽤 안전해 보인다.

기존 객체를 직접 수정하지 않고, 새로운 상태 객체를 생성해 참조를 교체하기 때문이다.

---

하지만 이 코드는 내부적으로 보면 결국 아래와 같은 흐름이다.

```text
1. 현재 state 읽기
2. 새로운 state 생성
3. 새로운 state 저장
```

즉:

```text
read
→ modify
→ write
```

구조다.

---

이 말은 결국 여러 thread가 동시에 상태를 변경하는 경우 아래와 같은 상황이 발생할 수 있다는 의미다.

```text
Thread A : count = 0 읽음
Thread B : count = 0 읽음

Thread A : count = 1 저장
Thread B : count = 1 저장
```

즉:

> 증가 연산은 두 번 수행되었지만 실제 결과는 한 번만 반영된다.

immutable state를 사용하더라도 read-modify-write 경쟁 상태 자체가 사라지는 것은 아니었던 것이다.

---

실제로 아래와 같은 간단한 예제를 실행해보면 이를 확인할 수 있다.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.MutableStateFlow

data class UiState(
    val count: Int = 0
)

val state = MutableStateFlow(UiState())

suspend fun increment() {
    repeat(1000) {
        state.value = state.value.copy(
            count = state.value.count + 1
        )
    }
}

fun main() = runBlocking {

    val job1 = launch(Dispatchers.Default) {
        increment()
    }

    val job2 = launch(Dispatchers.Default) {
        increment()
    }

    joinAll(job1, job2)

    println(state.value.count)
}
```

처음에는 최종 결과가 항상 `2000` 이 나올 것이라고 생각했었다.

하지만 실제로는:

```text
1734
1861
1922
...
```

처럼 실행 시점에 따라 값이 달라질 수 있었다.

즉:

> immutable state와 atomic reference만으로는
> read-modify-write 전체가 보호되지 않는다.

---

이 지점에서 `MutableStateFlow.update {}` 와 CAS(compare-and-swap) 기반 retry 구조가 왜 필요한지도 조금씩 이해되기 시작했다.

처음에는 CAS라는 개념이 꽤 복잡하게 느껴졌었다.

하지만 핵심은 생각보다 단순했다.

CAS는 결국:

> "내가 읽었던 상태가 아직 최신 상태인가?"

를 확인하는 과정에 가까웠다.

---

예를 들어 아래와 같은 상황을 생각해보자.

```text
Thread A : state = A 읽음
Thread B : state = A 읽음
```

이후 Thread B가 먼저 새로운 상태를 저장한다.

```text
A → C
```

그런데 Thread A는 여전히 이전 상태인 `A` 를 기준으로 새로운 상태를 저장하려고 시도한다.

이때 CAS는:

```text
현재 state가 아직도 A인가?
```

를 확인한다.

즉 사실상:

> "현재 참조가 내가 읽었던 참조와 동일한가?"

를 비교하는 것에 가깝다.

---

immutable state 구조에서는 기존 객체를 직접 수정하지 않고 새로운 객체를 생성해 참조를 교체한다.

즉:

```text
참조가 바뀌었다
=
누군가 상태를 변경했다
```

는 의미가 된다.

그래서 CAS는:

```text
expected reference
vs
current reference
```

를 비교하는 방식으로 동작하게 된다.

---

만약 이미 다른 thread가 state를 변경했다면 저장에 실패하고:

- 최신 state를 다시 읽고
- 새로운 state를 다시 계산하고
- 다시 저장을 시도한다.

즉:

```text
최신 상태를 기준으로 다시 계산하는 retry 구조
```

에 가까운 방식이다.

---

그리고 이런 CAS 역시 결국 CPU 레벨 atomic instruction 기반으로 동작한다.

대표적으로 CPU는:

- compare
- conditional swap

과정을 중간에 다른 thread가 끼어들 수 없도록 atomic하게 수행할 수 있는 instruction을 제공한다.

즉 CAS 역시 atomic operation의 한 종류라고 볼 수 있다.

---

그래서 `MutableStateFlow.update {}` 는 아래와 같은 형태로 사용된다.

```kotlin
state.update {
    it.copy(count = it.count + 1)
}
```

그리고 이 방식은 내부적으로 CAS 기반 retry를 통해 lost update를 방어한다.

---

하지만 이후 프로젝트를 다시 돌아보며 느낀 것은:

> 결국 중요한 것은
> "동시에 write 하는 thread가 실제로 존재하는가"
> 에 더 가까운 문제라는 점이었다.

single writer 구조라면 생각보다 CAS까지 필요한 상황은 많지 않을 수도 있다.

반대로 여러 thread가 동시에 동일 state를 수정할 수 있는 구조라면:

- immutable state
- atomic reference

만으로는 충분하지 않고,

```text
read
→ modify
→ write
```

전체를 어떻게 안전하게 보호할 것인가가 더 중요한 문제가 된다.

---

처음에는:

> "StateFlow는 atomic하니까 완전히 안전하겠지"

라고 생각했었다.

하지만 이후에는 조금 관점이 달라졌다.

중요한 것은 단순히 immutable state를 사용하는가보다:

> 현재 구조에서
> state를 write 하는 주체가 몇 개인가

에 더 가까운 문제였던 것 같다.

---

물론 여기서 또 하나의 흥미로운 문제가 남는다.

> "immutable state는 정말 immutable한가?"

라는 문제다.
