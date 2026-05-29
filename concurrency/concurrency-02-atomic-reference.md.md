# Atomic을 바라보는 관점이 바뀌다

처음 atomic을 접했을 때는 단순히:

```cpp
std::atomic<int>
```

처럼 숫자를 thread-safe 하게 증가시키는 low-level 기술이라고만 생각했다.

즉:

- lock 없이 증가 연산을 수행하고
- race condition을 방어하는

정도의 역할로 이해하고 있었다.

---

하지만 이후 Kotlin의 `StateFlow` 와 immutable state 패턴을 접하며 atomic을 바라보는 관점이 조금 달라졌다.

중요한 것은:

> "값 자체를 수정하는 것"
>
> 이 아니라
>
> "상태 객체의 참조를 교체하는 것"

이라는 점이었다.

---

예를 들어 Kotlin에서는 아래와 같은 immutable state 패턴을 자주 사용한다.

```kotlin
data class UiState(
    val count: Int = 0,
    val loading: Boolean = false
)

private val _state = MutableStateFlow(UiState())
```

여기서 상태를 변경할 때는 기존 객체를 직접 수정하지 않는다.

대신:

```kotlin
_state.value = _state.value.copy(
    count = 1
)
```

처럼 새로운 상태 객체를 생성하고, 상태 참조를 교체한다.

---

이 방식의 중요한 점은:

- 기존 객체는 변경되지 않고
- reader는 항상 완성된 상태 객체를 읽으며
- 상태 변경 중간 과정이 외부에 노출되지 않는다는 점이다.

즉:

```text
기존 상태 수정
```

이 아니라,

```text
새 상태 생성
→ 참조 교체
```

방식으로 상태를 관리한다.

---

이 지점에서 atomic은 단순 숫자 증가 연산 이상의 의미로 다가오기 시작했다.

atomic은:

> "상태를 안전하게 교체하는 방법"

으로도 활용될 수 있었기 때문이다.

---

물론 이후에는:

- 여러 writer가 동시에 상태를 변경하는 문제
- read-modify-write 과정의 경쟁 상태
- CAS(compare-and-swap)

같은 더 복잡한 주제들도 등장하게 된다.

하지만 적어도 atomic을 처음 이해하는 단계에서는:

> "상태를 직접 수정하지 않고,
> immutable state와 참조 교체 방식으로 관리할 수 있다"

는 관점 자체가 꽤 인상적으로 다가왔던 것 같다.