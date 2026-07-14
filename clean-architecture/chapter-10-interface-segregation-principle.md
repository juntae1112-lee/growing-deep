# ISP는 사용하지 않는 계약에 의존하지 않게 하는 원칙이다

> 인터페이스 분리 원칙은 인터페이스를 작게 만들라는 말이 아니라, 필요 없는 변경 이유를 같이 물려받지 않게 하라는 말이다.

인터페이스 분리 원칙은 다른 원칙들보다 비교적 직관적으로 느껴졌다.

클라이언트는 자신이 사용하지 않는 메서드에 의존하면 안 된다.

처음에는 이 말이 단순하게 들렸다.

"필요 없는 함수가 있으면 인터페이스를 나누면 되는 것 아닌가?"

큰 틀에서는 맞다고 생각한다.

내가 어떤 인터페이스를 상속받아 구현해야 하는데, 나에게 필요 없는 함수가 있다면 그 인터페이스는 너무 많은 것을 요구하고 있는 것이다.

그리고 그 필요 없는 함수가 바뀌면, 실제로 사용하지 않는 나에게도 변경 영향이 전파된다.

ISP는 그 불필요한 전파를 막기 위한 원칙에 가깝다.

---

# 필요 없는 함수도 변경 이유가 된다

처음에는 사용하지 않는 함수가 있어도 큰 문제처럼 보이지 않을 수 있다.

빈 구현을 두면 되기 때문이다.

```kotlin
interface RouteEngine {
    fun prepare()
    fun requestRoute(request: RouteRequest): RouteResult
}
```

어떤 엔진은 `prepare`가 필요할 수 있다.

하지만 어떤 엔진은 준비할 것이 없을 수 있다.

그럼 이렇게 구현할 수 있다.

```kotlin
class LocalRouteEngine : RouteEngine {
    override fun prepare() {
        // do nothing
    }

    override fun requestRoute(request: RouteRequest): RouteResult {
        // local route search
    }
}
```

겉으로는 큰 문제가 없어 보인다.

호출자도 같은 방식으로 호출할 수 있다.

```kotlin
engine.prepare()
val result = engine.requestRoute(request)
```

하지만 문제는 `prepare`가 바뀔 때 드러난다.

`prepare`에 새로운 조건이 추가될 수 있다.

반환값이 필요해질 수 있다.

실패 처리가 추가될 수 있다.

호출 순서가 더 엄격해질 수 있다.

그 순간 `prepare`를 실제로 사용하지 않는 구현체도 영향을 받는다.

필요 없는 함수였지만, 변경 이유는 같이 물려받은 것이다.

---

# 인터페이스는 기능 묶음이 아니라 의존성 묶음이다

인터페이스를 설계할 때 기능이 비슷하다는 이유로 한곳에 묶기 쉽다.

예를 들어 경로 엔진과 관련된 기능을 모두 하나의 인터페이스에 넣을 수 있다.

```kotlin
interface RouteEngine {
    fun initialize()
    fun prepare()
    fun requestRoute(request: RouteRequest): RouteResult
    fun cancel()
    fun release()
}
```

이렇게 하면 한눈에 보기에는 편하다.

경로 엔진이 할 수 있는 일이 모두 모여 있기 때문이다.

하지만 모든 클라이언트가 이 모든 함수를 필요로 하는 것은 아니다.

어떤 클라이언트는 경로 탐색만 필요하다.

어떤 클라이언트는 취소만 필요하다.

어떤 클라이언트는 생명주기 관리만 필요하다.

그런데 하나의 큰 인터페이스에 모두 묶여 있으면, 클라이언트는 자신이 사용하지 않는 함수까지 함께 의존하게 된다.

인터페이스는 단순히 기능을 모아둔 목록이 아니다.

그 인터페이스를 아는 쪽이 무엇의 변경에 영향을 받을지를 결정하는 의존성 묶음이다.

그래서 인터페이스가 커질수록 편해지는 것이 아니라, 영향 범위도 같이 커진다.

---

# 필요한 역할별로 나누어야 한다

ISP 관점에서는 인터페이스를 사용하는 쪽이 실제로 필요한 역할을 기준으로 나누는 편이 낫다.

예를 들어 모든 기능을 하나의 `RouteEngine`에 넣기보다 역할을 나눌 수 있다.

```kotlin
interface RouteSearcher {
    fun requestRoute(request: RouteRequest): RouteResult
}

interface RouteCanceler {
    fun cancel()
}

interface RouteEngineLifecycle {
    fun initialize()
    fun release()
}
```

이렇게 나누면 클라이언트는 자신에게 필요한 계약만 알면 된다.

경로 탐색만 필요한 쪽은 `RouteSearcher`만 의존한다.

```kotlin
class RouteControl(
    private val routeSearcher: RouteSearcher
) {
    fun search(request: RouteRequest): RouteResult {
        return routeSearcher.requestRoute(request)
    }
}
```

취소만 필요한 쪽은 `RouteCanceler`만 의존한다.

생명주기를 관리하는 쪽은 `RouteEngineLifecycle`만 의존한다.

그리고 실제 구현체는 필요한 인터페이스들을 함께 구현할 수 있다.

```kotlin
class DefaultRouteEngine :
    RouteSearcher,
    RouteCanceler,
    RouteEngineLifecycle {

    override fun initialize() {
        // initialize engine
    }

    override fun requestRoute(request: RouteRequest): RouteResult {
        // search route
    }

    override fun cancel() {
        // cancel route search
    }

    override fun release() {
        // release engine
    }
}
```

중요한 것은 구현 클래스가 여러 인터페이스를 구현할 수 있다는 점이다.

인터페이스를 나눈다고 해서 구현체도 반드시 여러 개로 쪼개야 하는 것은 아니다.

하나의 클래스가 여러 역할을 실제로 수행할 수 있다면, 그 클래스가 여러 인터페이스를 구현하면 된다.

하지만 사용하는 쪽은 자신에게 필요한 역할만 의존하게 된다.

---

# 나눈 인터페이스는 누군가 연결해줘야 한다

인터페이스를 역할별로 나누면 한 가지 질문이 생긴다.

"그럼 실제 구현체는 누가 넣어주지?"

예를 들어 실제 구현체는 하나일 수 있다.

```kotlin
class DefaultRouteEngine :
    RouteSearcher,
    RouteCanceler,
    RouteEngineLifecycle {
    // ...
}
```

하지만 사용하는 쪽은 각자 필요한 인터페이스만 받는다.

```kotlin
class RouteControl(
    private val routeSearcher: RouteSearcher
)

class RouteCancelButtonHandler(
    private val routeCanceler: RouteCanceler
)

class RouteEngineManager(
    private val lifecycle: RouteEngineLifecycle
)
```

그렇다면 바깥에서 실제 구현체를 밀어 넣어줘야 한다.

```kotlin
val engine = DefaultRouteEngine()

val routeControl = RouteControl(routeSearcher = engine)
val cancelHandler = RouteCancelButtonHandler(routeCanceler = engine)
val manager = RouteEngineManager(lifecycle = engine)
```

이것도 DI다.

DI 프레임워크를 써야만 DI인 것은 아니다.

중요한 것은 사용하는 쪽이 직접 구현체를 만들지 않는 것이다.

```kotlin
class RouteControl {
    private val engine = DefaultRouteEngine()
}
```

이렇게 되면 `RouteControl`은 실제로 `requestRoute`만 필요하더라도 `DefaultRouteEngine` 전체를 알게 된다.

인터페이스를 나눠도 사용하는 쪽이 구체 구현체를 직접 만들면 효과가 줄어든다.

좋은 구조에서는 역할이 나뉜다.

```text
사용하는 쪽
→ 자신에게 필요한 interface만 안다.

구현하는 쪽
→ 필요한 interface들을 구현한다.

조립하는 쪽
→ 실제 구현체를 알고, 필요한 곳에 넣어준다.
```

같은 구현체를 주입하더라도, 사용하는 쪽이 바라보는 인터페이스는 다를 수 있다.

이것이 ISP와 DI가 함께 쓰이는 이유라고 느꼈다.

ISP는 필요한 계약만 보게 한다.

DI는 그 계약에 실제 구현체를 연결한다.

즉 인터페이스를 나누는 것만으로 끝나는 것이 아니다.

나눈 인터페이스를 어디서 조립할 것인지도 함께 정해야 한다.

---

# ISP는 구현체를 위한 원칙이 아니라 사용자 쪽을 위한 원칙이다

처음에는 인터페이스 분리 원칙을 구현 클래스 입장에서 생각하기 쉽다.

"이 클래스가 구현해야 할 함수가 너무 많다."

이것도 맞는 말이다.

하지만 더 중요한 것은 사용하는 쪽이다.

클라이언트가 사용하지 않는 함수까지 알게 되면, 그 함수의 변경에 영향을 받을 수 있다.

예를 들어 `RouteControl`이 실제로는 탐색만 하면 되는데 큰 `RouteEngine` 전체를 알고 있다면 어떨까.

```kotlin
class RouteControl(
    private val routeEngine: RouteEngine
)
```

`RouteControl`은 `requestRoute`만 사용하더라도, 타입상으로는 `initialize`, `prepare`, `cancel`, `release`까지 모두 아는 구조가 된다.

이때 `cancel`의 의미가 바뀌거나 `prepare`의 계약이 바뀌면, `RouteControl`이 직접 사용하지 않더라도 영향 범위 안에 들어올 수 있다.

반대로 `RouteSearcher`만 의존하면 영향 범위가 줄어든다.

```kotlin
class RouteControl(
    private val routeSearcher: RouteSearcher
)
```

이제 `RouteControl`은 탐색 계약만 안다.

취소, 생명주기, 준비 과정의 변경은 `RouteControl`의 관심사가 아니다.

이것이 ISP의 핵심이라고 느꼈다.

> 사용하는 쪽이 필요로 하는 만큼만 알게 하라.

---

# LSP와 ISP는 이어져 있다

리스코프 치환 원칙을 보면서 이런 생각을 했다.

어떤 자식만 `prepare`가 필요하다면, 부모에 `prepare`를 올리면 LSP는 지킬 수 있지 않을까?

호출자는 모든 자식을 같은 방식으로 사용할 수 있기 때문이다.

하지만 그 순간 안 쓰는 자식도 `prepare`를 구현해야 한다.

```text
LSP 문제
→ 특정 자식만 다르게 사용해야 한다.

잘못된 해결
→ 부모 인터페이스에 특정 자식에게만 필요한 함수를 올린다.

새로운 문제
→ 필요 없는 자식도 그 함수를 구현해야 한다.
→ 필요 없는 클라이언트도 그 함수를 알게 된다.
→ ISP 위반 가능성이 생긴다.
```

이 연결이 중요하다고 느꼈다.

하나의 원칙을 지키려고 인터페이스를 넓히면, 다른 원칙을 어길 수 있다.

그래서 부모 계약에 함수를 추가할 때는 항상 물어봐야 한다.

```text
이 함수는 모든 구현체에게 자연스러운가?
이 함수는 이 인터페이스를 사용하는 모든 클라이언트에게 필요한가?
```

두 질문 중 하나라도 아니라면 인터페이스를 분리해야 할 가능성이 높다.

---

# 내가 얻은 결론

ISP는 인터페이스를 무조건 작게 쪼개라는 원칙은 아니었다.

작게 만드는 것이 목적이 아니라, 필요 없는 변경 이유에 의존하지 않게 만드는 것이 목적이다.

나에게 필요 없는 함수가 인터페이스에 포함되어 있다면, 나는 그 함수를 호출하지 않아도 그 함수의 변경에 영향을 받을 수 있다.

그래서 인터페이스는 기능이 비슷한지를 기준으로 묶는 것이 아니라, 사용하는 쪽이 필요로 하는 역할을 기준으로 나누어야 한다.

그리고 실제 구현체는 여러 인터페이스를 함께 구현할 수 있다.

분리되어야 하는 것은 반드시 클래스가 아니라 계약이다.

그 계약은 사용하는 쪽에는 작게 보이고, 조립하는 쪽에서 실제 구현체와 연결된다.

나는 인터페이스 분리를 "함수를 작게 나누는 일"이 아니라, 내가 책임지지 않아도 되는 변경 이유를 내 경계 밖으로 밀어내는 일로 이해했다.

내 기준으로 ISP는 이렇게 정리된다.

> 사용하지 않는 함수는 단순히 불필요한 함수가 아니다.  
> 나와 상관없는 변경 이유가 들어오는 통로다.

그래서 좋은 인터페이스는 많은 기능을 보여주는 인터페이스가 아니다.

필요한 쪽에 필요한 약속만 보여주는 인터페이스다.
