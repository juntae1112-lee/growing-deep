# 인터페이스는 누가 소유해야 할까

> 다형성을 쓴다고 항상 경계가 생기는 것은 아니다. 중요한 것은 인터페이스를 누가 소유하느냐다.

예전에는 교체 가능성을 만들 때 상속과 다형성을 먼저 떠올렸다.

어떤 구현을 나중에 바꿀 수 있어야 한다면, 공통 부모를 만들고 여러 구현체가 그것을 상속하면 된다고 생각했다.

이 생각이 완전히 틀린 것은 아니다.

하지만 지금 돌아보면 충분하지 않았다.

다형성을 사용했더라도, 인터페이스의 위치가 잘못되면 의존성 방향은 여전히 잘못될 수 있다.

---

# Google Auto SDK 개발 때의 구조

Google Auto SDK를 사용해 탐색 기능을 개발할 때였다.

당시 구조는 대략 다음과 같았다.

```text
UI
→ RouteControl
→ RouteEngine
```

관리자 요구사항은 RouteEngine을 다른 것으로 바꿀 수 있게 만드는 것이었다.

당시에는 RouteEngine을 상속 구조로 만들고, 구현체를 교체할 수 있게 하면 된다고 생각했다.

```text
RouteEngine
├─ GoogleRouteEngine
└─ OtherRouteEngine
```

이것도 분명 다형성은 맞다.

하지만 지금 다시 보면 좋은 구조는 아니었다.

문제는 `RouteControl`이 `RouteEngine`을 알고 있다는 점이었다.

```text
RouteControl
→ RouteEngine
```

RouteControl은 비즈니스 로직에 가까운 영역이었다.

RouteEngine은 외부 엔진 또는 데이터 세부사항에 가까운 영역이었다.

그렇다면 이 구조는 결국 비즈니스 로직이 데이터 세부사항을 아는 구조였다.

```text
Business Logic
→ Data Detail
```

겉으로는 엔진을 갈아끼울 수 있어 보인다.

하지만 교체 가능성의 기준은 RouteEngine 쪽에 있었다.

RouteControl이 필요한 동작을 정의한 것이 아니라, RouteEngine이 제공하는 모양에 RouteControl이 맞춰지는 구조였다.

---

# 지금이라면 다르게 설계했을 것이다

지금이라면 RouteControl 쪽, 즉 비즈니스 로직 쪽에 필요한 동작을 interface로 정의했을 것 같다.

```text
RouteControl
→ RouteSearchPort
← GoogleRouteEngineAdapter
```

RouteControl은 `RouteSearchPort`만 안다.

Google Auto SDK 기반 구현체는 그 계약을 구현한다.

```cpp
class RouteSearchPort {
public:
    virtual RouteResult searchRoute(const RouteRequest& request) = 0;
    virtual void cancelRoute() = 0;
    virtual ~RouteSearchPort() = default;
};
```

Google SDK를 사용하는 구현체는 이렇게 될 수 있다.

```cpp
class GoogleRouteEngineAdapter : public RouteSearchPort {
public:
    RouteResult searchRoute(const RouteRequest& request) override {
        // Google Auto SDK 호출
        // Google 응답을 RouteResult로 변환
    }

    void cancelRoute() override {
        // Google RouteEngine cancel 호출
    }
};
```

RouteControl은 외부 엔진의 구체 타입을 모른다.

그저 자신에게 필요한 계약만 안다.

실제 구현은 바깥쪽에서 들어온다.

```text
RouteControl
→ RouteSearchPort
← GoogleRouteEngineAdapter
```

이렇게 했어야 교체 가능성이 코드로 강제된다.

---

# 책바퀴에서 다시 만난 같은 문제

책바퀴 프로젝트에서도 비슷한 문제가 있었다.

내 동네를 프로필에 업데이트해야 하는 상황이었다.

처음에는 `neighborhood` feature가 `user` feature의 프로필 기능을 직접 가져다 쓸 수 있었다.

```text
feature/neighborhood
→ feature/user
```

빠르게 구현하려면 쉬운 방법이다.

어차피 동네 정보는 프로필에 저장되니, 프로필을 가져다 업데이트하면 된다.

하지만 이렇게 하면 feature 간 직접 의존이 생긴다.

`neighborhood`는 `user`의 내부 프로필 구조를 알아야 한다.

나중에 동네 저장 위치가 프로필 필드에서 별도 테이블로 바뀌면, `neighborhood` feature도 함께 흔들릴 수 있다.

그래서 구조를 바꿨다.

`core/domain`에 `MyNeighborhoodRepository`라는 계약을 두고, `user/data`가 그 계약을 구현하도록 했다.

```text
feature/neighborhood
→ core/domain/MyNeighborhoodRepository
← feature/user/data/ProfileRepositoryImpl
```

이제 `neighborhood`는 `user`의 내부 구현을 모른다.

그저 "내 동네를 저장한다"는 계약만 안다.

실제로는 `ProfileRepositoryImpl`이 `profiles.neighborhood`를 업데이트할 수 있다.

하지만 그 사실은 `neighborhood` feature가 몰라도 된다.

이 구조가 좋은 이유는 명확하다.

- `neighborhood`는 `user` feature를 직접 의존하지 않는다.
- `user/data`는 `core/domain`의 계약을 구현한다.
- 저장 방식이 바뀌어도 계약이 유지되면 `neighborhood`는 유지될 수 있다.
- feature 간 경계가 코드로 강제된다.

---

# 다형성을 어디에 두느냐가 중요하다

이 경험을 통해 다형성을 쓴다고 항상 좋은 설계가 되는 것은 아니라는 생각을 하게 됐다.

RouteEngine 상속 구조도 다형성이다.

하지만 그 다형성의 기준이 외부 엔진 쪽에 있으면, 비즈니스 로직은 여전히 외부 엔진의 모양에 묶인다.

중요한 것은 다형성을 사용했는지가 아니라:

> interface의 소유자가 누구인가

이다.

외부 엔진이 제공하는 추상화를 비즈니스 로직이 따르게 만들면, 의존성은 여전히 바깥쪽을 향한다.

반대로 비즈니스 로직이 필요한 동작을 interface로 정의하고, 외부 엔진이 그 interface를 구현하게 하면 의존성 방향이 바뀐다.

```text
잘못된 방향
Business Logic → External Engine

더 나은 방향
Business Logic → Business Interface ← External Engine
```

객체지향의 힘은 구현체를 여러 개 만드는 데서 끝나지 않는다.

그 구현체들이 어떤 계약을 따르게 할 것인지, 그리고 그 계약을 누가 소유할 것인지에서 나온다.

---

# 내가 얻은 결론

이제 interface를 볼 때 단순히 "테스트하기 좋다"거나 "구현체를 바꿀 수 있다" 정도로만 보지 않게 됐다.

더 먼저 묻게 된다.

```text
이 interface는 누가 소유해야 하는가?
이 계약은 사용하는 쪽의 필요를 표현하는가?
아니면 구현체의 모양을 노출하고 있는가?
이 의존성은 안쪽을 향하고 있는가?
```

다형성은 구현체를 여러 개 만드는 기술이 아니다.

적어도 아키텍처 관점에서는, 안정적인 정책이 세부사항에 끌려가지 않도록 의존성 방향을 제어하는 기술에 가깝다.

그리고 그 방향을 결정하는 가장 중요한 질문은 이것이다.

> 이 interface는 누가 소유해야 하는가?
