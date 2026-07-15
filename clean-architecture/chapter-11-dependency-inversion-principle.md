# 흐름은 바깥으로 가도, 의존성은 안쪽을 향해야 한다

> 의존성 역전 원칙은 실행 흐름과 코드 의존성을 분리해서, 변경 가능성이 큰 구현이 안정적인 정책을 흔들지 못하게 하는 원칙이다.

의존성 역전 원칙을 처음 읽을 때는 조금 헷갈렸다.

구체화된 클래스에 의존하지 말고 추상화에 의존하라.

구체 클래스는 변경될 가능성이 크기 때문에 그것을 상속받거나 직접 의존하지 말라.

이 말 자체는 이해된다.

구체 구현은 자주 바뀐다.

API가 바뀔 수 있고, DB가 바뀔 수 있고, 외부 SDK가 바뀔 수 있다.

그런 구체 구현에 중요한 로직이 직접 의존하면, 구체 구현의 변경이 중요한 로직까지 전파된다.

그런데 처음에는 이런 생각도 들었다.

"이거 결국 인터페이스를 분리하라는 말 아닌가?"

하지만 다시 생각해보니 DIP의 핵심은 인터페이스를 만든다는 사실 자체가 아니었다.

중요한 것은 의존성의 방향이었다.

---

# 제어 흐름과 의존성 방향은 다를 수 있다

DIP를 이해하는 데 가장 도움이 된 문장은 이것이었다.

> 제어 흐름과 코드 의존성은 반대가 될 수 있다.

책바퀴의 data layer를 생각하면 이해하기 쉽다.

사용자가 동네 기준으로 책 목록을 조회한다고 해보자.

실행 흐름은 대략 이렇게 흘러간다.

```text
UI
→ UseCase
→ Repository 구현체
→ API
```

실제로 데이터를 가져오려면 결국 바깥의 API나 DB로 나가야 한다.

그러니 실행 흐름만 보면 domain이 data를 호출하는 것처럼 보인다.

하지만 코드 의존성까지 그렇게 가면 안 된다.

```text
Domain → Data 구현체 → API
```

이렇게 되면 domain이 data layer의 구체 구현을 알게 된다.

API 응답 형식이 바뀌거나, 통신 방식이 바뀌거나, 저장소가 바뀌면 domain도 같이 흔들린다.

그래서 의존성 방향을 뒤집는다.

```text
Domain
└─ Repository Interface

Data
└─ Repository Impl
   └─ API / DB
```

실행될 때는 UseCase가 Repository를 통해 데이터를 얻는다.

하지만 코드 의존성은 data가 domain의 interface를 바라본다.

```text
제어 흐름
UI → Domain → Data → API

코드 의존성
UI → Domain
Data → Domain
```

이것이 내가 이해한 의존성 역전이다.

흐름은 데이터를 얻기 위해 바깥으로 나가지만, 코드 의존성은 비즈니스 규칙을 보호하기 위해 안쪽을 향한다.

---

# 왜 안쪽 정책을 향해야 할까

의존성이 안쪽 정책을 향해야 하는 이유는 안쪽이 더 고귀해서가 아니다.

안쪽 정책이 더 안정적이어야 하기 때문이다.

책바퀴에서 domain은 앱의 규칙을 담는다.

```text
사용자는 동네를 가진다.
동네 기준으로 책을 조회한다.
책을 등록한다.
책 상세 정보를 본다.
프로필을 수정한다.
```

이런 규칙도 물론 바뀔 수 있다.

하지만 API 응답 형식이나 DB 구현보다 자주 바뀌어야 하는 것은 아니다.

반대로 data layer는 변경 가능성이 크다.

```text
REST API를 쓸지
GraphQL을 쓸지
Kakao API를 쓸지
다른 API를 쓸지
Room DB를 쓸지
DataStore를 쓸지
캐시 정책을 어떻게 가져갈지
```

이런 것들은 언제든 바뀔 수 있다.

그러니 변경 가능성이 큰 data layer가 안정적이어야 할 domain을 흔들면 안 된다.

DIP는 이 흔들림을 막는다.

```text
변경 가능성이 큰 구현
→ 변경 가능성이 낮은 정책에 맞춘다
```

즉 의존성은 더 안정적인 쪽을 향해야 한다.

내 기준으로 DIP는 이렇게 정리된다.

> 변경 가능성이 큰 구현이 변경 가능성이 낮은 정책을 흔들지 못하게 의존성 방향을 뒤집는 원칙이다.

---

# OCP와 DIP는 같은 말이 아니다

여기서 OCP와 DIP가 조금 겹쳐 보일 수 있다.

의존성 방향을 잘 뒤집으면 새로운 구현을 추가하기 쉬워진다.

예를 들어 domain이 `BookRepository` interface만 알고 있다면, data layer에서는 여러 구현체를 추가할 수 있다.

```text
KakaoBookRepository
NaverBookRepository
LocalBookRepository
MockBookRepository
```

이때 domain은 덜 수정된다.

그러니 DIP를 적용하면 OCP를 지키기 쉬워지는 경우가 많다.

하지만 두 원칙이 묻는 질문은 다르다.

OCP는 이렇게 묻는다.

```text
새로운 요구사항이 들어왔을 때,
기존 코드를 덜 수정하고 확장할 수 있는가?
```

DIP는 이렇게 묻는다.

```text
코드 의존성이 안정적인 정책을 향하고 있는가?
변경 가능성이 큰 구현이 안정적인 정책을 흔들고 있지는 않은가?
```

즉 OCP는 확장과 수정의 문제다.

DIP는 의존성 방향의 문제다.

DIP는 OCP를 돕는 강력한 방법이 될 수 있다.

하지만 DIP가 곧 OCP는 아니다.

이 구분을 하고 나니 DIP가 조금 더 선명해졌다.

---

# interface는 누구의 요구사항인가

DIP에서 interface가 중요하긴 하다.

하지만 interface를 만들었다고 자동으로 DIP가 되는 것은 아니다.

중요한 것은 그 interface가 누구의 요구사항을 표현하느냐다.

예를 들어 data layer가 자신이 제공하는 방식대로 interface를 만들었다고 해보자.

```kotlin
interface KakaoBookApiClient {
    suspend fun requestKakaoBooks(
        regionCode: String,
        page: Int
    ): KakaoBookResponse
}
```

이 interface를 domain이 그대로 사용한다면 이름만 추상화일 뿐이다.

domain은 여전히 카카오 API의 사정에 맞춰져 있다.

DIP에서 필요한 interface는 domain이 필요로 하는 계약이어야 한다.

```kotlin
interface BookRepository {
    suspend fun getBooksByNeighborhood(
        neighborhoodId: NeighborhoodId
    ): List<Book>
}
```

domain은 이 계약만 안다.

```kotlin
class GetBooksUseCase(
    private val bookRepository: BookRepository
) {
    suspend operator fun invoke(
        neighborhoodId: NeighborhoodId
    ): List<Book> {
        return bookRepository.getBooksByNeighborhood(neighborhoodId)
    }
}
```

그리고 data layer가 이 계약을 구현한다.

```kotlin
class KakaoBookRepository(
    private val api: KakaoBookApi
) : BookRepository {

    override suspend fun getBooksByNeighborhood(
        neighborhoodId: NeighborhoodId
    ): List<Book> {
        return api.getBooks(neighborhoodId.value)
            .map { it.toDomain() }
    }
}
```

이 구조에서는 domain이 카카오 API를 모른다.

카카오 API가 바뀌어도 domain의 규칙은 덜 흔들린다.

중요한 것은 interface의 존재가 아니라 interface의 소유권이다.

> interface는 구체 구현의 메뉴판이 아니라, 안정적인 정책이 필요로 하는 계약이어야 한다.

---

# UI도 바깥쪽 세부사항이다

처음에는 data layer에 대해서는 이해가 쉬웠다.

data는 변경 가능성이 크고, domain이 data 구현체를 알면 안 된다.

그런데 한 가지 질문이 생겼다.

"그럼 UI와 presentation은 왜 의존성 역전을 잘 하지 않을까?"

생각해보니 이유는 간단했다.

UI는 원래 바깥쪽이다.

그리고 일반적인 흐름에서는 UI가 domain을 사용한다.

```text
UI → Presentation → Domain
```

코드 의존성도 이미 안쪽을 향한다.

그래서 보통은 추가로 뒤집을 필요가 없다.

문제는 domain이 UI를 알 때 생긴다.

```text
Domain → Toast
Domain → Android View
Domain → Compose State
Domain → Navigation
```

이렇게 되면 UI 변경이 domain까지 전파된다.

UI도 자주 바뀌는 세부사항이다.

```text
XML View에서 Compose로 바뀔 수 있다.
모바일 UI와 태블릿 UI가 나뉠 수 있다.
웹으로 확장될 수 있다.
디자인 시스템이 바뀔 수 있다.
화면 플로우가 바뀔 수 있다.
```

반면 domain의 규칙은 이런 UI 변화에 흔들리면 안 된다.

```text
사용자는 동네를 가진다.
동네 기준으로 책을 조회한다.
책을 등록한다.
```

이 규칙은 UI가 어떻게 바뀌든 유지되어야 한다.

그래서 UI와 data는 둘 다 바깥쪽 세부사항이라고 볼 수 있다.

```text
UI → Domain ← Data
```

UI는 domain을 사용하고, data는 domain의 계약을 구현한다.

둘 다 domain을 향한다.

domain은 둘 모두를 모른다.

---

# 도메인 레이어를 잡는 사람이 중요한 이유

이 관점에서 보면 domain layer를 잡는 사람의 역할이 정말 중요하다.

domain은 단순히 중간에 있는 코드가 아니다.

제품의 규칙을 코드로 정의하는 곳이다.

domain은 이런 질문에 답해야 한다.

```text
우리 서비스에서 사용자는 무엇을 할 수 있는가?
어떤 상태가 유효한가?
어떤 흐름이 허용되는가?
어떤 정책을 지켜야 하는가?
외부 API나 UI가 바뀌어도 유지되어야 하는 규칙은 무엇인가?
```

domain이 약하면 정책은 바깥으로 새어나간다.

UI가 정책을 가진다.

data가 정책을 가진다.

API 응답 구조가 그대로 앱의 규칙이 된다.

화면마다 같은 판단을 반복한다.

변경이 생기면 어디를 고쳐야 할지 알기 어려워진다.

반대로 domain이 안정적이면 UI와 data는 바뀌어도 된다.

UI는 표현 방식을 바꾼다.

data는 가져오는 방식을 바꾼다.

하지만 제품의 핵심 규칙은 domain에 남는다.

그래서 domain layer를 잡는 사람은 기술만 잘해서는 부족하다.

제품의 규칙을 이해해야 하고, 변경 가능성을 봐야 하고, 책임의 경계를 잡아야 한다.

결국 여기서도 질문은 같다.

> 이것은 누구의 책임인가?

---

# 회사의 Controller 구조로 다시 보면

MVC 기준으로 보면 domain은 Controller보다 Model에 가깝다.

Controller는 입력을 받고 흐름을 조율하는 역할에 가깝고, 비즈니스 규칙은 Model이나 Domain에 가까운 쪽에 있어야 한다.

하지만 우리 회사에서 말하는 Controller는 이미 규칙까지 담당하는 경우가 많다.

이름은 Controller지만 실제로는 기능의 중심 객체에 가깝다.

```text
Controller
├─ 이벤트 수신
├─ 엔진 호출
├─ 상태 관리
├─ 정책 판단
├─ 예외 처리
├─ 화면 데이터 생성
└─ 외부 모듈 송신
```

이런 구조에서 "Controller는 무조건 나쁘다"라고 말하는 것은 현실과 맞지 않는다.

중요한 것은 이름이 아니다.

그 안에 어떤 변경 이유들이 섞여 있는지가 중요하다.

Controller가 규칙을 가진다면, 최소한 그 규칙과 세부 구현은 분리되어야 한다.

예를 들어 Controller가 feature의 중심이라면 이렇게 생각할 수 있다.

```text
Feature Controller
→ feature 흐름의 owner

Feature Policy
→ feature 판단 규칙

Feature State
→ feature 상태의 source of truth

Feature Port / Interface
→ 외부 모듈과의 계약
```

즉 회사 구조에서는 Clean Architecture의 용어를 그대로 가져오기보다, 현재 구조 안에서 같은 질문을 던져야 한다.

```text
이 Controller가 가져야 할 규칙은 무엇인가?
이 Controller 밖으로 밀어내야 할 세부사항은 무엇인가?
이 Controller 안에서도 다시 나눌 수 있는 책임은 무엇인가?
```

Controller가 모든 세부사항을 직접 알면 변경 이유가 너무 많아진다.

그러면 결국 Controller는 변경 가능성이 큰 구현들에게 흔들린다.

DIP 관점에서 중요한 것은 Controller라는 이름을 없애는 것이 아니다.

중요한 정책이 더 불안정한 세부사항에 끌려가지 않게 만드는 것이다.

---

# 내가 얻은 결론

의존성 역전 원칙은 단순히 interface를 만들라는 말이 아니었다.

제어 흐름과 코드 의존성을 분리해서, 안정적인 정책을 보호하는 원칙이었다.

실행 흐름은 바깥 구현으로 나갈 수 있다.

데이터를 가져오려면 API로 가야 하고, 저장하려면 DB로 가야 한다.

하지만 코드 의존성까지 그쪽으로 향하면 안 된다.

코드 의존성은 더 안정적인 정책을 향해야 한다.

책바퀴 기준으로 보면 UI와 data는 둘 다 바깥쪽 세부사항이다.

UI는 domain을 사용하고, data는 domain의 계약을 구현한다.

domain은 UI와 data를 모른다.

그래야 UI가 바뀌고, API가 바뀌고, DB가 바뀌어도 비즈니스 규칙이 덜 흔들린다.

내 기준으로 DIP는 이렇게 정리된다.

> 흐름은 바깥으로 가도 된다.  
> 하지만 의존성은 안정적인 정책을 향해야 한다.

그리고 interface는 구체 구현의 편의를 드러내는 메뉴판이 아니라, 안정적인 정책이 필요로 하는 계약이어야 한다.
