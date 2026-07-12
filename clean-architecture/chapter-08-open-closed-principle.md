# OCP는 변경 가능성이 큰 곳과 작은 곳을 나누는 원칙이다

> 개방 폐쇄 원칙은 변경이 필요한 곳은 열어두고, 변경이 번지면 안 되는 곳은 닫아두는 원칙이다.

개방 폐쇄 원칙을 처음 보면 문장이 조금 이상하게 느껴진다.

확장에는 열려 있고, 수정에는 닫혀 있어야 한다.

처음에는 이 말이 잘 와닿지 않았다.

코드는 요구사항이 바뀌면 당연히 수정되는 것 아닌가.

새 기능을 추가하려면 기존 코드도 어느 정도 바뀌는 것 아닌가.

그런데 책바퀴 구조를 생각하면서 조금 감이 왔다.

OCP는 코드를 절대 수정하지 말라는 말이 아니었다.

변경이 일어날 가능성이 큰 부분과, 상대적으로 안정적으로 지켜야 하는 부분을 나누라는 말에 가까웠다.

---

# 변경 가능성이 큰 부분은 바깥에 있다

책바퀴에서 변경 가능성이 큰 부분을 생각해보면 먼저 `data` 레이어가 떠오른다.

API는 언제든 바뀔 수 있다.

처음에는 REST API를 쓰다가 나중에는 다른 형태의 API를 쓸 수도 있다.

응답 필드가 바뀔 수도 있고, 인증 방식이 바뀔 수도 있고, 캐시 정책이 바뀔 수도 있다.

지도도 마찬가지다.

처음에는 카카오 지도를 쓸 수 있다.

나중에는 네이버 지도로 바뀔 수도 있고, 구글 지도로 바뀔 수도 있고, 자체 지도 SDK를 붙일 수도 있다.

이런 것들은 중요한 구현이지만, 비즈니스의 본질은 아니다.

책바퀴의 핵심은 어떤 API를 쓰느냐가 아니다.

사용자가 책을 등록하고, 동네를 기준으로 책을 발견하고, 관심 있는 책을 살펴보고, 사용자 정보를 관리하는 흐름이다.

API와 지도 SDK는 이 흐름을 가능하게 하는 세부사항에 가깝다.

그래서 이런 부분은 변경에 열려 있어야 한다.

```text
API 구현
지도 SDK
DB 구현
캐시 정책
외부 로그인
위치 제공 방식
```

이런 것들은 바뀔 가능성이 크다.

그러니 바뀌어도 되는 곳으로 밀어내야 한다.

---

# 반대로 비즈니스 로직은 상대적으로 안정적이다

반면 domain 레이어의 비즈니스 로직은 상대적으로 변경 확률이 낮다.

물론 비즈니스 로직도 바뀔 수 있다.

하지만 API 응답 형식이나 지도 SDK보다 자주 바뀌어야 하는 부분은 아니다.

예를 들어 책바퀴에서 이런 규칙은 상대적으로 안정적이다.

```text
사용자는 자신의 동네를 가진다.
책은 특정 동네와 연결될 수 있다.
동네를 기준으로 책 목록을 볼 수 있다.
사용자는 프로필 정보를 수정할 수 있다.
```

이런 규칙은 앱의 중심에 가깝다.

그래서 이 부분이 API 변경이나 지도 SDK 변경 때문에 흔들리면 구조가 약한 것이다.

예를 들어 domain이 카카오 API 응답 모델을 직접 알고 있다면 어떨까.

```kotlin
data class UserNeighborhood(
    val kakaoAddressName: String,
    val kakaoRegionCode: String
)
```

처음에는 빠르게 만들 수 있다.

하지만 카카오 API가 바뀌거나 다른 지도 API로 바꾸는 순간 domain도 같이 바뀐다.

이 구조에서는 바깥의 세부사항이 안쪽의 비즈니스 로직까지 침투한 것이다.

OCP 관점에서 보면 닫혀 있어야 할 domain이 외부 변경에 열려 있는 상태다.

---

# RouteEngine을 교체 가능하게 만들고 싶었던 경험

이 원칙은 예전에 RouteEngine을 교체 가능하게 만들고 싶었던 경험과도 연결된다.

당시 구조는 대략 이런 방향이었다.

```text
UI
→ RouteControl
→ RouteEngine
```

요구사항은 RouteEngine을 나중에 다른 엔진으로 바꿀 수 있게 만드는 것이었다.

그때 나는 RouteEngine을 상속 구조와 팩토리 구조로 만들면 된다고 생각했다.

```text
RouteControl
→ RouteEngineFactory
→ RouteEngine
   ├─ DefaultRouteEngine
   ├─ PeriodicRerouteEngine
   ├─ UserRerouteEngine
   └─ OptionRouteEngine
```

기본 탐색, 주기적 재탐색, 사용자 재탐색, 옵션 탐색처럼 동작을 나누면 확장 가능하다고 생각했다.

새로운 탐색 방식이 필요하면 새로운 엔진을 추가하고, 팩토리에서 상황에 맞는 엔진을 생성하면 되기 때문이다.

이것도 어느 정도는 OCP에 가까운 생각이었다.

새로운 동작을 추가할 수 있는 확장 지점과 생성 책임을 분리하려고 했기 때문이다.

하지만 지금 다시 보면 부족한 부분이 있었다.

문제는 팩토리를 썼는지 여부가 아니었다.

더 중요한 문제는 RouteControl이 여전히 RouteEngine을 알고 있었고, RouteEngine의 사양에 맞춰 함수를 구현하고 있었다는 점이다.

RouteControl은 비즈니스 흐름에 가까운 역할인데, 실제 구조에서는 Control이 자신에게 필요한 계약을 정의하기보다 Engine이 제공하는 방식에 맞춰지고 있었다.

```text
RouteControl
→ RouteEngine
→ Engine이 제공하는 함수에 맞춰 구현
```

이렇게 되면 RouteEngine을 팩토리로 교체할 수 있게 만든 것처럼 보여도, 실제로는 RouteControl이 엔진의 변화에 계속 영향을 받는다.

엔진의 종류가 늘거나, 엔진 호출 방식이 바뀌거나, 탐색 결과 형식이 바뀌면 RouteControl도 같이 흔들릴 수 있다.

즉 확장에는 어느 정도 열려 있었지만, RouteControl이 수정에 닫혀 있지는 않았다.

지금이라면 다르게 설계할 것 같다.

RouteControl이 RouteEngine의 사양에 맞춰지는 것이 아니라, RouteControl이 필요한 동작을 interface로 정의했을 것이다.

```kotlin
interface RouteSearcher {
    fun search(request: RouteSearchRequest): RouteSearchResult
}
```

RouteControl은 이 계약만 사용한다.

```text
RouteControl
→ RouteSearcher
```

그리고 실제 RouteEngine은 이 계약을 구현한다.

```text
DefaultRouteEngine → RouteSearcher 구현
NewRouteEngine     → RouteSearcher 구현
MockRouteEngine    → RouteSearcher 구현
```

이렇게 하면 RouteControl은 엔진의 내부 사양에 맞춰지지 않는다.

그저 자신이 필요한 "경로를 탐색할 수 있다"는 계약만 안다.

새로운 엔진이 추가되어도 RouteControl의 핵심 흐름은 덜 흔들린다.

이때 비로소 확장에는 열려 있고, 수정에는 닫힌 구조에 가까워진다.

---

# domain은 자신이 필요한 계약만 알아야 한다

그래서 책바퀴에서 더 나은 방향은 domain이 외부 구현을 직접 아는 것이 아니라, 자신에게 필요한 계약만 가지는 것이다.

예를 들어 domain은 카카오 API를 알 필요가 없다.

domain이 알아야 하는 것은 "현재 위치로 동네를 얻을 수 있다"는 개념이다.

```kotlin
interface NeighborhoodResolver {
    suspend fun resolveCurrentNeighborhood(): Neighborhood
}
```

또는 사용자의 동네를 변경해야 한다면 이런 계약을 가질 수 있다.

```kotlin
interface UserNeighborhoodRepository {
    suspend fun updateNeighborhood(neighborhood: Neighborhood)
}
```

domain은 이 계약을 사용해서 비즈니스 흐름을 만든다.

```kotlin
class UpdateMyNeighborhoodUseCase(
    private val neighborhoodResolver: NeighborhoodResolver,
    private val userNeighborhoodRepository: UserNeighborhoodRepository
) {
    suspend fun update() {
        val neighborhood = neighborhoodResolver.resolveCurrentNeighborhood()
        userNeighborhoodRepository.updateNeighborhood(neighborhood)
    }
}
```

여기서 중요한 점은 domain이 카카오 API도, 네이버 지도도, 서버 응답 형식도 모른다는 것이다.

domain은 오직 자신의 언어로 말한다.

```text
현재 위치의 동네를 구한다.
사용자의 동네를 변경한다.
```

이렇게 하면 지도 API가 바뀌어도 domain의 문장은 바뀌지 않는다.

변경은 data 레이어의 구현체에서 일어난다.

```text
KakaoNeighborhoodResolver
NaverNeighborhoodResolver
GoogleNeighborhoodResolver
```

새로운 지도 제공자가 필요하면 새로운 구현체를 추가하면 된다.

domain의 핵심 흐름은 그대로 둘 수 있다.

이것이 확장에는 열려 있고, 수정에는 닫혀 있다는 말의 실무적인 의미라고 느꼈다.

---

# 의존성 방향을 제어해야 변경 전파를 막을 수 있다

OCP를 이해하다 보면 결국 의존성 방향으로 이어진다.

만약 domain이 data를 직접 바라보면 어떻게 될까.

```text
domain
→ KakaoMapApi
→ UserApiResponse
→ RoomEntity
```

이 구조에서는 data의 변경이 domain으로 바로 올라온다.

API가 바뀌면 domain이 바뀐다.

DB가 바뀌면 domain이 바뀐다.

지도 SDK가 바뀌면 domain이 바뀐다.

즉 변경 가능성이 큰 부분이 변경 가능성이 작은 부분을 흔든다.

반대로 책바퀴에서 내가 가져가고 싶은 방향은 이렇다.

```text
ui → domain
data → domain
```

domain은 중심에 있다.

ui도 domain을 바라보고, data도 domain을 바라본다.

data는 domain이 정의한 interface를 구현한다.

```text
domain
├─ UseCase
├─ Entity
└─ Interface

data
└─ Interface 구현
```

이 구조에서는 data가 바뀌어도 domain의 핵심 규칙은 덜 흔들린다.

변경 가능성이 큰 구현은 바깥에 있고, 안정적으로 유지하고 싶은 정책은 안쪽에 있다.

OCP는 결국 이 경계를 만들라는 말로 이해할 수 있다.

---

# 확장에는 열려 있다는 말

확장에 열려 있다는 것은 새로운 요구사항이 들어왔을 때 기존의 안정적인 코드를 계속 열어 수정하지 않아도 된다는 뜻이다.

예를 들어 책바퀴에서 지도 제공자를 바꿔야 한다고 해보자.

나쁜 구조에서는 domain이나 use case 안에 지도 API 코드가 직접 들어 있다.

그러면 지도 API를 바꾸는 순간 use case도 수정해야 한다.

반면 domain이 `NeighborhoodResolver`라는 계약만 알고 있다면, 새로운 구현체를 추가할 수 있다.

```text
KakaoNeighborhoodResolver 추가
NaverNeighborhoodResolver 추가
MockNeighborhoodResolver 추가
```

테스트용 구현체도 만들기 쉬워진다.

서버 API가 준비되지 않았을 때도 fake 구현으로 domain 흐름을 검증할 수 있다.

즉 확장 포인트가 생긴다.

새로운 구현을 추가할 수 있는 자리가 생긴다.

---

# 수정에는 닫혀 있다는 말

수정에 닫혀 있다는 것은 코드를 절대 수정하지 않는다는 뜻이 아니다.

수정되면 안 되는 중요한 정책이 외부 구현 변경 때문에 흔들리지 않는다는 뜻이다.

책바퀴에서 닫혀 있어야 할 것은 이런 부분이다.

```text
사용자의 동네를 변경하는 흐름
책 목록을 동네 기준으로 조회하는 규칙
사용자 프로필을 갱신하는 유스케이스
도메인 모델의 의미
```

반대로 열려 있어도 되는 것은 이런 부분이다.

```text
카카오 API를 호출하는 방식
네이버 API를 호출하는 방식
지도 SDK 연결 방식
서버 응답을 domain 모델로 변환하는 방식
로컬 캐시 구현
```

이렇게 보면 OCP는 "수정을 하지 말자"가 아니라 "수정이 일어나는 위치를 제한하자"에 가깝다.

---

# 내가 얻은 결론

OCP를 처음에는 조금 추상적인 원칙으로 느꼈다.

하지만 책바퀴 구조에 대입하니 훨씬 선명해졌다.

변경 가능성이 큰 부분은 data 레이어에 많다.

API, 지도 SDK, DB, 외부 로그인, 위치 제공 방식은 언제든 바뀔 수 있다.

반면 domain의 비즈니스 로직은 상대적으로 안정적이어야 한다.

그래서 domain이 data를 직접 알면 안 된다.

domain은 자신이 필요한 계약을 정의하고, data가 그 계약을 구현해야 한다.

이렇게 해야 새로운 API나 지도 SDK가 추가되어도 domain의 핵심 흐름은 덜 흔들린다.

RouteEngine을 상속 구조와 팩토리 구조로 나누려 했던 경험도 결국 같은 문제였다.

나는 확장 가능한 구조를 만들려고 했다.

기본 탐색, 주기적 재탐색, 사용자 재탐색, 옵션 탐색을 나누고, 팩토리에서 필요한 엔진을 만들면 새로운 탐색 방식을 추가하기 쉬워질 것이라고 생각했다.

그 판단 자체가 완전히 틀렸다고 생각하지는 않는다.

확장 지점을 만들려는 시도였기 때문이다.

하지만 당시에는 무엇이 닫혀 있어야 하는지까지는 충분히 보지 못했다.

닫혀 있어야 했던 것은 RouteControl의 핵심 흐름이었다.

열려 있어야 했던 것은 RouteEngine의 구체 구현과 생성 방식이었다.

지금이라면 RouteControl이 RouteEngine의 사양에 맞춰 구현되게 하기보다, RouteControl이 필요한 탐색 계약을 정의하고 엔진이 그 계약을 구현하게 만들 것 같다.

```text
RouteControl
→ RouteSearcher

RouteEngine
→ RouteSearcher 구현
```

이렇게 보면 OCP는 단순히 상속 구조를 만들거나 구현체를 늘리는 원칙이 아니다.

상속을 쓰는 것도, 팩토리를 쓰는 것도, interface를 쓰는 것도 그 자체로 좋은 설계가 되지는 않는다.

중요한 것은 그 구조를 왜 쓰는지 아는 것이다.

무엇을 교체 가능하게 만들고 싶은지.

무엇이 바뀌어도 기존 흐름이 흔들리지 않게 하고 싶은지.

어떤 변경을 열어두고, 어떤 정책을 닫아두고 싶은지.

그 의도를 모르면 패턴을 써도 구조는 쉽게 흐려진다.

더 중요한 질문은 이것이다.

> 무엇을 열어두고, 무엇을 닫아둘 것인가?

결국 OCP는 내가 계속 고민하던 질문과 이어진다.

> 변경은 어디까지 전파되어야 하는가?

OCP는 이 질문에 이렇게 답하는 원칙이라고 생각한다.

> 자주 바뀌는 세부사항은 열어두고, 덜 바뀌어야 하는 비즈니스 정책은 닫아둔다.

그래서 개방 폐쇄 원칙은 단순히 확장 가능한 코드를 만들라는 말이 아니다.

변경 가능성이 큰 부분이 안정적인 부분을 흔들지 못하게 의존성 방향을 제어하는 원칙이다.
