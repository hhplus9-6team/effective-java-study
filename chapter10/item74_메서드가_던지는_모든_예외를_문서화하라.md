# item 74. 메서드가 던지는 모든 예외를 문서화하라
> * 메서드가 **던질 수 있는 모든 예외는 반드시 문서화**해야 한다.
> * 특히 **checked exception**은 `throws` 선언 + Javadoc에 **언제 발생하는지 명확히 설명**해야 한다.
> * **unchecked exception(RuntimeException)** 도 가능하면 Javadoc의 `@throws`로 문서화하는 것이 좋다.
> * 예외도 API의 **중요한 계약(contract)** 이다.

---

## 1. 왜 예외 문서화가 중요한가?

* 예외는 단순한 에러가 아니라 **메서드 사용 조건과 실패 시 의미**를 나타낸다.
* 문서화되지 않은 예외는

    * 호출자 입장에서 “이 메서드는 언제 실패하는지” 알 수 없고
    * 방어 코드나 복구 로직을 작성하기 어렵다.
* 특히 public 메서드는 반드시 예외까지 문서화해야한다.
---

## 2. 예외 문서화의 기준 정리

* 반드시 문서화해야 하는 것

    * 모든 checked exception
    * 메서드의 전제조건 위반을 의미하는 RuntimeException
* 문서화하지 않아도 되는 것

    * 프로그래밍 오류에 가까운 내부 버그 수준의 RuntimeException
* 문서화 시 유의점

    * 예외 이름만 나열하지 말 것
    * “언제, 어떤 조건에서” 발생하는지 명확히 설명할 것

---

## 3. Checked Exception 문서화 규칙

* 반드시 다음 두 가지를 모두 지켜야 한다.

    * 메서드 시그니처에 `throws` - 예외는 항상 따로따로 선언한다.
    * Javadoc에 `@throws` 설명

### 좋은 예

```java
/**
 * 지정한 ID의 사용자를 조회한다.
 *
 * @param userId 사용자 ID
 * @return 사용자
 * @throws UserNotFoundException 해당 ID의 사용자가 존재하지 않을 경우
 */
public User findById(Long userId) throws UserNotFoundException {
    ...
}
```

* 예외 이름만 나열하지 말고 **발생 조건을 문장으로 설명**해야 한다.

---

## 4. Unchecked Exception도 문서화하라

* RuntimeException이라고 해서 문서화하지 않아도 되는 것은 아니다.
* 특히 **메서드의 전제조건을 위반했을 때 발생하는 예외**는 꼭 문서화해야 한다.
* 단, 메서드 throws에는 선언하지 않는다.(checked exception만 선언)

### 좋은 예

```java
/**
 * 두 수를 나눈다.
 *
 * @param a 피제수
 * @param b 제수 (0이면 안 됨)
 * @return 나눈 결과
 * @throws IllegalArgumentException b가 0인 경우
 */
public int divide(int a, int b) {
    if (b == 0) {
        throw new IllegalArgumentException("b must not be zero");
    }
    return a / b;
}
```

---

## 5. 구현 세부 예외를 문서에 노출하지 말 것

* 내부 구현 때문에 발생하는 예외를 그대로 드러내면 안 된다.

### 나쁜 예

```java
/**
 * @throws NullPointerException
 * @throws IndexOutOfBoundsException
 */
public String getName(int index) {
    return list.get(index).getName();
}
```

* 이건 구현체(ArrayList, 내부 필드 접근)를 그대로 노출한 것
* API 사용자는 **왜 예외가 발생하는지** 알 수 없다.

### 좋은 예

```java
/**
 * @param index 사용자 인덱스
 * @return 사용자 이름
 * @throws IllegalArgumentException index가 유효 범위를 벗어난 경우
 */
public String getName(int index) {
    if (index < 0 || index >= list.size()) {
        throw new IllegalArgumentException("Invalid index");
    }
    return list.get(index).getName();
}
```

* 의미 중심의 예외로 변환하고 문서화한다.

---
## 7. 메서드의 예외가 중복되면 class의 설명에 추가하자.
```java
/***
 * 여기에 작성
  */
public interface Collection<E> extends Iterable<E> {
 ```
* 대표 예시: `Collection`

---
# 추가) 실무에서 어떻게 적용해볼 수 있을까 고민해본거

## 1. 책(Item 74)의 전제부터 정리
실무(Spring + RuntimeException only)에서는

* 메서드 호출자가 “Java 메서드 사용자”가 아니라
* **HTTP API 사용자 / 다른 팀 / 프론트 / 외부 시스템**
  인 경우가 대부분.

그래서 **문서화 위치도 달라져야 한다.**

---

## 2. 실무 정답: “누가 이 실패를 소비하느냐”로 결정한다

### A. Controller / 외부 API → **API 문서가 정답**

* 대상: 프론트엔드, 외부 서비스, 앱
* 문서화 위치:

    * OpenAPI(Swagger)
    * API 명세서(Confluence, Notion, README)
* 문서화 내용:

    * HTTP 상태
    * 에러 코드
    * 발생 조건
    * 재시도 가능 여부

**이 경우 Javadoc은 거의 의미 없다.**

---

### B. Application Service / UseCase → **Javadoc이 여전히 유효**

* 대상: 같은 서버 내 다른 개발자, 미래의 나
* 문서화 목적:

    * “이 유스케이스를 호출할 때 어떤 전제조건이 필요한가”
    * “어떤 비즈니스 실패가 발생 가능한가”

이 경우는 **Item 74 스타일을 축약해서 적용**한다.

#### 권장 스타일

```java
/**
 * 주문을 생성한다.
 *
 * <p>다음 경우 주문 생성에 실패할 수 있다:
 * <ul>
 *   <li>재고가 부족한 경우</li>
 *   <li>이미 처리된 멱등 키인 경우</li>
 * </ul>
 *
 * @throws OrderDomainException 도메인 규칙 위반 시
 */
public OrderId placeOrder(PlaceOrderCommand command) {
    ...
}
```

* RuntimeException이라도
* **“어떤 실패 시나리오가 존재하는지”를 문서화**
* 예외 클래스 이름은 최소화

---

### C. Domain Model / Aggregate → **Javadoc은 거의 계약 그 자체**

* 대상: 도메인을 사용하는 개발자
* 이 레벨은 오히려 책과 가장 잘 맞는다.

```java
/**
 * 재고를 차감한다.
 *
 * @param quantity 차감 수량 (1 이상)
 * @throws InsufficientStockException
 *         현재 재고가 부족한 경우
 */
public void decrease(int quantity) {
    if (stock < quantity) {
        throw new InsufficientStockException();
    }
}
```

---
## RuntimeException + Enum 조합으로 사용하는 경우

### 1. 구조
* CommonErrorCode (인증/권한/검증/시스템)
* OrderErrorCode, ProductErrorCode, CouponErrorCode 처럼 도메인별

이렇게 나누면:
* 한 enum에 다 때려넣는 지옥을 피하고
* 팀/모듈 단위로 책임이 생김

```java
public interface ErrorCode {
    String code();          // 예: ORD-001
    HttpStatus status();    // 예: CONFLICT
    String message();       // 기본 메시지
    boolean retryable();    // 재시도 가능 여부
}
```
```java
public enum OrderErrorCode implements ErrorCode {
    OUT_OF_STOCK("ORD-001", HttpStatus.CONFLICT, "재고가 부족합니다.", false),
    DUPLICATE_REQUEST("ORD-002", HttpStatus.CONFLICT, "중복 요청입니다.", false);
    // fields, ctor, getters...
}
```
```java
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.message());
        this.errorCode = errorCode;
    }

    public ErrorCode errorCode() { return errorCode; }
}
```
```java
throw new BusinessException(OrderErrorCode.OUT_OF_STOCK);
```
## javadocs 템플릿
ErrorCode를 문자열로 직접 쓰면(예: ORD-001) 변경 시 반영 안됨. 그래서 enum 상수를 참조하는 방식이 좋다.
```java
/**
 * 주문을 생성한다.
 *
 * <h3>Failure cases</h3>
 * <ul>
 *   <li>{@link OrderErrorCode#OUT_OF_STOCK}: 재고가 부족한 경우</li>
 *   <li>{@link OrderErrorCode#DUPLICATE_REQUEST}: 동일 멱등키 중복 요청</li>
 * </ul>
 */
public OrderId placeOrder(PlaceOrderCommand command) { ... }

```

## QnA
* 예외 문서화시 반드시 유의해야할 점(1가지)