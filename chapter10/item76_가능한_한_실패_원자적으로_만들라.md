# item76. 가능한 한 실패 원자적으로 만들라

## 1. 실패 원자성이란?

실패 원자성이란 **메서드 호출이 실패하더라도, 객체의 상태는 호출 이전과 동일하게 유지되어야 한다**는 의미입니다.

즉, 성공하거나 아무 일도 없었던 것처럼 돌아가야 한다는 성질입니다.

이 개념은 트랜잭션의 원자성과 매우 유사하지만, 객체 단위의 상태 일관성에 초점을 두고 있습니다.

## 2. 왜 실패 원자성이 중요한가?

실패 원자성이 보장되지 않으면 예외 자체보다 예외 이후의 상태가 더 위험합니다.

- 예외가 발생했는데 객체 상태는 중간 단계로 깨져 있음
- 이후 호출에서 원인 파악이 어려운 버그 발생
- 한 번의 실패가 연쇄적인 오류로 이어짐
- 디버깅 난이도가 급격히 상승

## 3. 실패 원자성이 깨진 예제

```java
public void add(int value) {
    list.add(value);   // 상태 변경
    count++;           // 이후 예외가 발생하면 상태 불일치
}
```

- `list`에는 값이 들어갔지만
- `count`는 증가하지 않음

이 코드에서 `count++` 전에 예외가 발생하면 객체 상태가 논리적으로 불일치하게 됩니다.


## 4. 실패 원자성을 확보하는 방법

### 1) 불변 객체 사용

```java
public final class Money {
    private final BigDecimal amount;

    public Money add(BigDecimal value) {
        return new Money(amount.add(value));
    }
}
```

- 기존 객체 상태를 변경하지 않음
- 실패해도 원본 객체는 안전
- 실패 원자성이 자연스럽게 보장

### 2) 매개변수 유효성 검사를 먼저 수행

```java
public void withdraw(int amount) {
    if (amount < 0) {
        throw new IllegalArgumentException();
    }
    balance -= amount;
}
```

- 잘못된 입력은 상태 변경 전에 차단
- 매우 단순하지만 효과적인 방법

### 3) 상태 변경은 마지막에 수행하라

```java
public void push(E e) {
    ensureCapacity(); // 실패 가능
    elements[size++] = e; // 상태 변경
}
```

- 실패 가능성이 있는 작업을 먼저 수행하고 상태 변경은 맨 마지막에 함
- 이 패턴은 JDK 컬렉션에서도 자주 사용됨

### 4) 복구 코드 작성

```java
public void add(E e) {
    int oldSize = size;
    try {
        elements[size++] = e;
    } catch (Exception ex) {
        size = oldSize; // 복구
        throw ex;
    }
}
```

- 상태 변경 도중 예외 발생 시 이전 상태로 되돌리는 코드 직접 작성
- 코드 복잡도 증가
- 실수 가능성 증가

### 5) 임시 객체에서 작업 후 교체하기

```java
public void sort() {
    List<E> copy = new ArrayList<>(list);
    Collections.sort(copy); // 실패 가능
    list = copy;            // 성공 시에만 교체
}
```

- 실패하면 원본은 그대로 유지
- 성공하면 한 번에 상태 교체

## 5. 실패 원자성의 예외적인 경우

- 성능 때문에 실패 원자성을 포기하는 것이 합리적인 경우
- 객체가 실패 이후 다시 사용되지 않는 경우
- 복구 비용이 과도한 경우

실패 원자성은 API 품질을 높이기 위한 원칙이지, 모든 상황에서 강제해야 하는 절대 규칙은 아닙니다.

## 6. 결론

> 메서드가 실패하면 객체는 실패 이전 상태로 돌아가야 합니다.

- 불변 객체를 적극 사용하기
- 상태 변경은 마지막에 수행
- 필요하다면 임시 객체나 복구 전략 사용

실패 원자성을 지키는 코드는 예외 상황에서도 예측 가능하고 신뢰할 수 있는 코드를 만듭니다.


---

### 질문

Q. 실패 원자성이란 무엇인가요?

Q. 실패 원자성을 확보하는 방법 5가지를 나열해보세요.

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 76