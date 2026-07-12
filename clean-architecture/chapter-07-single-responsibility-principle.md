# SRP는 하나의 기능이 아니라 하나의 변경 이유를 말한다

> 하나의 책임이란 하나의 기능이 아니라, 하나의 actor에 의해 변경되는 이유다.

SRP를 처음 들으면 보통 이렇게 이해하기 쉽다.

하나의 클래스는 하나의 일만 해야 한다.

이 말도 틀리지는 않지만, 실무에서 바로 적용하려고 하면 애매해진다.

하나의 일이라는 게 어디까지일까.

함수 하나만 가져야 한다는 뜻일까.

클래스를 아주 작게 쪼개야 한다는 뜻일까.

Clean Architecture를 읽으면서 내가 다시 생각하게 된 부분은 `actor`였다.

SRP는 단순히 기능의 개수를 세는 원칙이 아니었다.

그 코드를 바꾸라고 요구하는 사람이 누구인지, 어떤 변경 이유를 따라 움직이는지를 보는 원칙에 가까웠다.

---

# actor는 사용자가 아니라 변경을 요구하는 쪽이다

처음에는 actor라는 말이 조금 헷갈렸다.

사용자를 말하는 건가.

기획자를 말하는 건가.

모듈을 말하는 건가.

지금은 이렇게 이해하고 있다.

> actor는 그 코드의 변경을 요구하는 이해관계자다.

꼭 사람이 아닐 수도 있다.

제품 사양일 수도 있고, 외부 시스템일 수도 있고, 특정 도메인의 정책일 수도 있다.

중요한 것은 이것이다.

> 이 코드는 누구의 요구 때문에 바뀌는가?

만약 하나의 코드가 서로 다른 이유로 계속 바뀐다면, 그 코드는 하나 이상의 책임을 가지고 있을 가능성이 크다.

---

# RouteExternal의 actor는 누구일까

예를 들어 경로 정보를 외부로 보내는 `RouteExternal` 같은 구조를 생각해볼 수 있다.

겉으로 보면 하나의 책임처럼 보인다.

```text
RouteExternal
→ 경로 정보를 외부로 보낸다
```

하지만 그 안에 여러 전송 대상이 있을 수 있다.

```text
RouteExternal
├─ SOME/IP
├─ NaviProvider
└─ 다른 외부 연동
```

이걸 무조건 나눠야 할까.

처음에는 "외부로 보낸다"는 기능이 같으니 하나로 묶어도 된다고 생각할 수 있다.

하지만 SRP 관점에서는 기능이 같은지가 아니라 변경 이유가 같은지를 봐야 한다.

SOME/IP가 바뀌는 이유는 차량 플랫폼이나 미들웨어 사양 때문일 수 있다.

NaviProvider가 바뀌는 이유는 내비 도메인 API나 서비스 정책 때문일 수 있다.

다른 외부 연동은 품질, 진단, 로그, 운영 정책 때문에 바뀔 수 있다.

겉으로는 모두 "외부로 보낸다"는 기능이지만, 변경을 요구하는 actor가 다를 수 있다.

그렇다면 하나의 클래스 안에 모두 넣는 순간 변경 이유가 섞인다.

```text
SOME/IP 변경
→ RouteExternal 수정

NaviProvider 변경
→ RouteExternal 수정

진단 정책 변경
→ RouteExternal 수정
```

이 구조에서는 `RouteExternal`이 점점 외부 전송과 관련된 모든 변경을 흡수하는 클래스가 된다.

그래서 더 나은 방향은 외부로 보내는 흐름과 각 전송 방식의 책임을 나누는 것이다.

```text
RouteExternal
├─ SomeIpRoutePublisher
├─ NaviProviderRoutePublisher
└─ DiagnosticRoutePublisher
```

`RouteExternal`은 경로 정보를 외부로 내보내는 흐름을 조율한다.

각 `Publisher`는 자신이 맡은 외부 계약과 전송 사양을 책임진다.

이렇게 나누면 단순히 클래스가 작아지는 것이 아니다.

변경 이유가 분리된다.

---

# 상태도 actor를 가진다

SRP를 클래스에만 적용하려고 하면 감이 잘 안 올 때가 있다.

그런데 상태를 설계할 때도 같은 관점이 필요하다.

예를 들어 `routeGuideState`라는 상태가 있다고 하자.

이 상태는 경로 안내 중 상태를 의미한다.

그렇다면 이 상태의 actor는 "현재 경로 안내"에 가깝다.

관심사는 이런 것들이다.

```text
안내 시작
안내 중
안내 일시정지
경로 이탈
목적지 도착
안내 종료
```

즉 `routeGuideState`는 현재 선택된 경로를 따라 안내가 어떻게 진행되고 있는지를 표현해야 한다.

그런데 여기에 `Searching` 같은 탐색 상태를 넣고 싶어질 수 있다.

```kotlin
sealed interface RouteGuideState {
    data object Idle : RouteGuideState
    data object Searching : RouteGuideState
    data object Guiding : RouteGuideState
    data object Paused : RouteGuideState
    data object Arrived : RouteGuideState
}
```

처음에는 자연스러워 보일 수 있다.

탐색이 끝나야 안내가 시작되기 때문이다.

하지만 다시 보면 탐색과 안내는 변경 이유가 다르다.

탐색은 경로를 찾는 과정이다.

안내는 선택된 경로를 따라가는 과정이다.

```text
RouteSearchState
→ 경로 요청
→ 탐색 중
→ 탐색 성공
→ 탐색 실패
→ 재시도

RouteGuideState
→ 안내 시작
→ 안내 중
→ 일시정지
→ 경로 이탈
→ 목적지 도착
→ 안내 종료
```

둘은 이어지는 흐름일 수는 있다.

하지만 같은 책임은 아니다.

`routeGuideState`에 탐색 상태를 넣는 순간, 이 상태는 더 이상 안내 상태만 의미하지 않는다.

이름은 Guide인데 실제로는 Search까지 표현한다.

이때부터 상태의 의미가 흐려진다.

그리고 상태의 의미가 흐려지면, 그 상태를 사용하는 코드들도 같이 흔들린다.

---

# 이름이 책임보다 좁으면 버그가 숨어든다

상태 이름이 `RouteGuideState`라면 읽는 사람은 이 값이 안내 상태만 표현한다고 기대한다.

하지만 그 안에 탐색 상태가 들어 있다면 이름과 실제 책임이 어긋난다.

이런 구조에서는 나중에 코드를 읽는 사람이 이런 질문을 하게 된다.

```text
Searching은 안내 중인가?
탐색 실패는 GuideState의 실패인가?
탐색 중일 때 현재 경로는 존재하는가?
안내 재시작과 재탐색은 같은 상태 전이인가?
```

이 질문들이 생긴다는 것 자체가 책임이 섞였다는 신호일 수 있다.

그래서 나는 이런 경우에는 상태를 분리하는 편이 더 맞다고 느낀다.

```kotlin
sealed interface RouteSearchState {
    data object Idle : RouteSearchState
    data object Searching : RouteSearchState
    data class Success(val routes: List<Route>) : RouteSearchState
    data class Failed(val reason: SearchError) : RouteSearchState
}
```

```kotlin
sealed interface RouteGuideState {
    data object Idle : RouteGuideState
    data class Guiding(val route: Route) : RouteGuideState
    data object Paused : RouteGuideState
    data object Arrived : RouteGuideState
}
```

이렇게 나누면 탐색 정책이 바뀔 때는 `RouteSearchState`를 보면 된다.

안내 정책이 바뀔 때는 `RouteGuideState`를 보면 된다.

각 상태가 어떤 actor의 요구로 바뀌는지 더 분명해진다.

---

# 더 넓은 상태가 필요하다면 이름도 넓어져야 한다

물론 항상 분리만이 답은 아니다.

제품에서 탐색과 안내를 하나의 사용자 흐름으로 보고 싶을 수도 있다.

예를 들어 "경로 세션"이라는 개념이 있다면 탐색 중, 안내 중, 도착 같은 상태를 하나의 상위 상태로 묶을 수 있다.

```kotlin
sealed interface RouteSessionState {
    data object Idle : RouteSessionState
    data object Searching : RouteSessionState
    data class Guiding(val route: Route) : RouteSessionState
    data object Arrived : RouteSessionState
}
```

이 경우에는 `RouteGuideState`라는 이름보다 `RouteSessionState`가 더 맞다.

책임이 넓어졌다면 이름도 넓어져야 한다.

이름은 단순한 표기가 아니다.

이 상태가 무엇을 책임지는지에 대한 계약이다.

---

# 내가 얻은 결론

SRP는 클래스를 작게 만들라는 말이 아니었다.

하나의 모듈이 하나의 actor에 대해서만 변경되도록 하라는 말이었다.

그래서 내가 설계할 때 던져야 할 질문도 조금 달라졌다.

```text
이 코드는 어떤 이유로 바뀌는가?

이 변경을 요구하는 actor는 누구인가?

이 상태는 무엇을 표현하는가?

이 이름은 실제 책임과 일치하는가?

서로 다른 변경 이유를 하나의 상태나 클래스에 섞고 있지는 않은가?
```

`RouteExternal`을 볼 때도 단순히 "외부로 보내는 기능"이라고 보면 부족하다.

각 외부 계약이 같은 actor를 가지는지 봐야 한다.

`routeGuideState`를 볼 때도 단순히 "경로와 관련된 상태"라고 보면 부족하다.

그 상태가 안내를 표현하는지, 탐색까지 포함하는지, 아니면 더 넓은 세션을 표현하는지 봐야 한다.

결국 SRP는 기능을 나누는 기술이라기보다, 변경 이유를 분리하는 기술에 가깝다.

그리고 좋은 이름은 그 책임의 경계를 드러내야 한다.

> 같은 기능인가보다, 같은 이유로 변경되는가가 더 중요하다.
