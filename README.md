# Growing Deep

개발하며 만난 문제를, 외운 답 대신 생각이 바뀐 과정으로 기록합니다.

이곳의 글은 특정 기술의 사용법을 빠르게 정리하기보다, "왜 이 구조가 필요한가?", "안전하다는 말은 정확히 어디까지인가?"를 실제 경험과 함께 따라갑니다. 좋은 설계는 대개 더 많은 것을 아는 일이 아니라, **변경과 상태가 일어나는 경계를 더 분명하게 보는 일**이라고 믿습니다.

## 읽는 순서

### 1. 변경의 경계

- [왜 우리는 책임을 분리해야 할까](boundary/boundary-01-why-we-separate-responsibilities.md)

  하나의 Controller에 역할이 쌓였던 경험을 통해, 책임 분리가 코드 정리가 아니라 변경 전파를 제한하는 설계라는 점을 살펴봅니다.

### 2. Feature 단위의 소유권

- [같은 사용자 의도는 하나의 Feature가 책임져야 한다](ownership/ownership-01-who-owns-common-flow.md)

  공용 패널을 여러 모듈이 나눠 쓰며 생긴 문제를 통해, 같은 사용자 의도는 feature가 하나의 흐름으로 끝까지 책임져야 한다는 생각을 다룹니다.
- [공통 정책은 누가 소유해야 할까](ownership/ownership-02-who-owns-common-policy.md)

  충전기 번개 아이콘처럼 여러 모듈이 같은 제품 사양을 따라야 할 때, 공통화해야 하는 것은 UI가 아니라 판단 기준과 변경 책임이라는 점을 다룹니다.

### 3. 상태 변경의 경계

- [왜 우리는 항상 Mutex부터 떠올릴까](concurrency/concurrency-01-atomic-operation.md.md)

  lock과 atomic operation의 차이, 그리고 lost update가 생기는 이유를 C++ 예제로 다룹니다.
- [Atomic을 바라보는 관점이 바뀌다](concurrency/concurrency-02-atomic-reference.md.md)

  숫자 하나의 원자적 증가를 넘어, immutable state와 참조 교체로 atomic을 이해해 봅니다.
- [왜 StateFlow의 update{} 가 필요할까](concurrency/concurrency-03-why-stateflow-needs-update.md)

  immutable state와 atomic reference만으로 해결되지 않는 read-modify-write 경쟁 상태를 따라갑니다.

## 이 저장소에서 다루는 질문

- 변경 이유가 다른 코드는 어디에서 갈라져야 할까?
- 상태를 안전하게 변경한다는 말은 무엇을 보장할까?
- single writer라는 구조는 왜 복잡도를 줄여 줄까?
- 기술의 편리한 추상화 뒤에는 어떤 제약이 남아 있을까?

## 기록 방식

각 글은 가능한 한 다음 흐름을 따릅니다.

1. 처음에 가졌던 직관
2. 그 직관이 충분하지 않았던 실제 상황
3. 문제를 다시 설명하는 핵심 개념
4. 이후 설계와 코드에서 달라진 판단

정답을 모아두기보다, 더 나은 질문에 도달한 과정을 남깁니다.
