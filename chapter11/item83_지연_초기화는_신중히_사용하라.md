# item83. 지연 초기화는 신중히 사용하라

## 지연 초기화란?

- 필드의 초기화 시점을 그 값이 처음 필요할 때까지 늦추는 기법
- 정적 필드와 인스턴스 필드 모두에 사용 가능
- 주로 최적화 용도로 사용되지만, 클래스와 인스턴스 초기화 때 발생하는 위험한 순환 문제를 해결하는 효과도 있음


## 최적화 용도가 아니라면 사용하지 말라

- **장점**: 클래스 혹은 인스턴스 생성 시의 초기화 비용은 줄어든다
- **단점**: 지연 초기화하는 필드에 접근하는 비용은 커진다
- **성능 영향 요소**:
  - 지연 초기화하려는 필드들 중 결국 초기화가 이뤄지는 비율
  - 실제 초기화에 드는 비용
  - 초기화된 각 필드를 얼마나 빈번히 호출하느냐

위 요소들에 따라 지연 초기화가 실제로는 성능을 더 느리게 할 수도 있다.



## 지연 초기화가 필요할 때

- 해당 클래스의 인스턴스 중 그 필드를 사용하는 인스턴스의 비율이 낮은 반면
- 그 필드를 초기화하는 비용이 크다면 지연 초기화가 제 역할을 해줄 것
- 적용 전후의 성능을 꼭 측정하자



## 멀티스레드 환경과 스레드 안전성

### 멀티스레드 환경에서의 지연 초기화
- 지연 초기화하는 필드를 둘 이상의 스레드가 공유한다면 어떤 형태로든 반드시 동기화해야 한다
- 그렇지 않으면 심각한 버그로 이어진다
- 이번 아이템에서 다루는 모든 초기화 기법은 스레드 안전하다


## 여러 가지 지연 초기화 기법

### 1. 일반적인 초기화

대부분의 상황에서는 일반적인 초기화가 지연 초기화보다 낫다

```java
// 인스턴스 필드
private final FieldType field1 = computeFieldValue();

// 정적 필드
private static final FieldType field1 = computeFieldValue();
```



---

### 2. synchronized 접근자 방식

지연 초기화가 초기화 순환성을 깨뜨릴 것 같으면`synchronized`를 단 접근자를 사용하자. 이 방법이 가장 간단하고 명확한 대안이다.

```java
private FieldType field;

private synchronized FieldType getField() {
    if (field == null)
        field = computeFieldValue();
    return field;
}
```

---

### 3. 지연 초기화 홀더 클래스 관용구 

성능 때문에 정적 필드를 지연 초기화해야 한다면 지연 초기화 홀더 클래스 관용구를 사용하자.

- 클래스는 클래스가 처음 쓰일 때 비로소 초기화된다는 특성을 이용한 관용구

```java
private static class FieldHolder {
    static final FieldType field = computeFieldValue();
}

private static FieldType getField() { 
    return FieldHolder.field; 
}
```

- `getField()`가 처음 호출되는 순간 `FieldHolder.field`가 처음 읽히면서, `FieldHolder` 클래스 초기화
- `getField()` 메서드가 필드에 접근하면서 동기화를 전혀 하지 않으니 성능이 느려질 거리가 전혀 없다
- 일반적인 VM은 오직 클래스를 초기화할 때만 필드 접근을 동기화한다
- 클래스 초기화가 끝난 후에는 VM이 동기화 코드를 제거하여, 그다음부터는 아무런 검사나 동기화 없이 필드에 접근하게 한다

---

### 4. 이중검사 관용구 

성능 때문에 인스턴스 필드를 지연 초기화해야 한다면 이중검사 관용구를 사용하라.

```java
private volatile FieldType field4;

private FieldType getField() {
    FieldType result = field4;
    if (result != null)    // 첫 번째 검사 (락 사용 안 함)
        return result;
    
    synchronized(this) {
        if (field4 == null) // 두 번째 검사 (락 사용)
            field4 = computeFieldValue();
        return field4;
    }
}
```

- 초기화된 필드에 접근할 때의 동기화 비용을 없애준다
- 필드의 값을 두 번 검사: 한 번은 동기화 없이, 두 번째는 동기화하여 검사
- 두 번째 검사에서도 필드가 초기화되지 않았을 때만 필드를 초기화
- 필드가 초기화된 후로는 동기화하지 않으므로 해당 필드는 반드시 `volatile`로 선언한다

#### `result`의 역할
- 필드가 이미 초기화된 상황에서는 이 필드를 딱 한 번만 읽도록 보장하는 역할
- 반드시 필요하진 않지만 성능을 높여주고, 저수준 동시성 프로그래밍에 표준적으로 적용되는 방법

---

### 5. 단일검사 관용구

반복해서 초기화해도 상관없는 인스턴스 필드를 지연 초기화해야 할 때가 있는데, 이런 경우라면 이중검사에서 두 번째 검사를 생략할 수 있다.

```java
private volatile FieldType field5;

private FieldType getField() {
    FieldType result = field5;
    if (result == null)
        field5 = result = computeFieldValue();
    return result;
}
```

- 초기화가 중복해서 일어날 수 있지만, 초기화 결과가 동일하다면 허용 가능한 경우에 사용

---

### 6. 짜릿한 단일검사 관용구

이 기법은 아주 이례적인 기법으로, 보통 거의 쓰지 않는다

- 모든 스레드가 필드의 값을 다시 계산해도 상관없고
- 필드의 타입이 `long`과 `double`을 제외한 다른 기본 타입이라면

단일검사의 필드 선언에서 `volatile` 한정자를 없애도 된다.

어떤 환경에서는 필드 접근 속도를 높여주지만, 초기화가 스레드당 최대 한 번 더 이뤄질 수 있다.

---

## QnA

Q1. 지연 초기화가 성능 향상에 도움이 되는 두 가지 조건은 무엇인가요?

Q2. 정적 필드와 인스턴스 필드를 지연 초기화할 때 각각 권장되는 기법은 무엇인가요?

---

## 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 83
