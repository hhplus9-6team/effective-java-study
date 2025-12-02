# ITEM 49. 매개변수가 유효한지 검사하라

## 메서드와 생성자

- 일반적으로 메서드와 생성자 대부분은 입력 매개변수 값이 특정 조건을 만족하길 바란다
- 인덱스 값은 음수여선 안 된다
- 객체 참조는 null이 아니어야 한다
- 위와 같은 제약은 반드시 문서화해야 하며, 메서드 몸체 시작 전에 검사(예외 처리)해야 한다

**오류는 가능한 한 빨리 발생한 곳에서 잡아야 한다.**

## 매개변수 검증이 이루어지지 않은 경우

1. **메서드가 수행되는 중간에 모호한 예외를 던지며 실패할 수 있다**

2. **더 나쁜 상황은 메서드가 잘 수행되지만 잘못된 결과를 반환한다**

3. **최악의 상황은 메서드는 수행됐지만, 객체를 손상시켜서 예측 불가능한 시점에 메서드와 관련 없는 오류를 낸다**

**실패 원자성(failure atomicity, Item 76)을 어기는 결과를 낳을 수 있다.**

## public과 protected 메서드 예외 처리

- 공개 API의 예외 처리는 문서화하라 (`@throws`, Item 74)
- `IllegalArgumentException`, `IndexOutOfBoundsException`, `NullPointerException` (Item 72)
- 매개변수 제약을 문서화한다면, 그 제약을 어겼을 때 발생하는 예외도 기술하라
- 발생한 예외와 API Docs와 던지기로 한 예외가 다르면 예외 번역(exception translate, Item 73) 관용구를 사용하라

```java
/**
 * (현재 값 mod m) 값을 반환한다.
 * 이 메서드는 항상 음이 아닌 BigInteger를 반환한다는 점에서 remainder 메서드와 다르다.
 *
 * @param m 계수(양수여야 한다.)
 * @return 현재 값 mod m
 * @throws ArithmeticException m이 0보다 작거나 같으면 발생한다.
 */
public BigInteger mod(BigInteger m) {
    if (m.signum() <= 0)
        throw new ArithmeticException("계수(m)는 양수여야 합니다. " + m);

    // 계산 수행
    ...
}
```

**모든 public 메서드에 적용되는 문서화를 하고 싶다면 클래스 수준에서 Javadoc을 작성하라.**

`m.signum()`은 m이 null일 때, `NullPointerException`을 던지지만 메서드에 언급이 없다. 해당 설명을 BigInteger 클래스 수준에서 기술했기 때문이다.

## null check

```java
this.strategy = Objects.requireNonNull(strategy, "전략");
```

**Java 7에 추가된 `java.util.Objects.requireNonNull` 메서드의 특징:**
- null check를 수동으로 하지 않아도 된다
- 예외 메시지를 지정할 수 있다
- 입력을 그대로 반환하므로, 값을 사용하면서 동시에 null check가 가능하다
- 반환값은 그냥 무시하고 순수한 null 검사 목적으로 어디서든 사용 가능하다

**Java 9의 Objects의 범위 검사 기능:**
- `checkFromIndexSize`, `checkFromToIndex`, `checkIndex`
- 예외 메시지 지정이 불가능하고, 리스트와 배열 전용이면서 닫힌 범위는 다루지 못한다


**메서드가 직접 사용하지는 않으나 나중에 쓰기 위해 저장하는 매개변수는 더 신경써야 한다.**

Item 20의 예제처럼 null check를 해주지 않으면, 클라이언트가 새롭게 생성한 List 인스턴스를 받는다. 돌려받는 List를 사용하고자 할 때 비로소 `NullPointerException`이 발생하므로 디버깅이 힘들어진다.

## 단언문(assert)

```java
private static void sort(long a[], int offset, int length) {
    assert a != null;
    assert offset >= 0 && offset <= a.length;
    assert length >= 0 && length <= a.length - offset;
    ... // 계산 수행
}
```

**단언문의 특징:**
- 단언문들은 자신이 단언한 조건들이 무조건 참이라고 선언한다
- 해당 메서드가 포함된 패키지를 클라이언트가 어떤 식으로 사용하든 상관없다
- 유효성 검사와 다르다
- 실패하면 `AssertionError`를 던진다
- 런타임에 아무런 효과도, 아무런 성능 저하도 없다

## 규칙의 예외

유효성 검사 비용이 지나치게 높거나, 실용성 떨어지거나, 암묵적으로 검사가 수행되는 경우

- `Collections.sort(List)`는 리스트 내 객체들이 모두 상호 비교가 되어야 하며, 정렬 과정에서 이루어지므로 비교 과정에서 `ClassCastException`이 발생한다
- 단, 암묵적 유효성 검사에 너무 의존하면 실패 원자성을 해칠 수 있다

## 결론

- **메서드나 생성자를 작성할 때면 그 매개변수들에 어떤 제약이 있을지 생각해야 한다**
- 그 제약들을 문서화하고, 메서드 코드 시작 부분에서 명시적으로 검사해야 한다
- 이런 습관을 반드시 기르도록 하자

---

## 질문

### Q1. 매개변수 검증이 이루어지지 않는 경우는?

### Q2. 단언문의 특징은 무엇인가요?
