# item62. 다른 타입이 적절하다면 문자열 사용을 피하라

문자열(String)은 자바에서 가장 흔히 사용하는 타입이자 가장 과도하게 남용되는 타입입니다.

문자열은 너무 유연해서 무엇이든 담을 수 있지만, 그 때문에 타입 안정성을 잃고, 도메인 의미가 흐려지며, 수많은 버그의 원인이 됩니다.

이 아이템에서는 문자열을 피해야 하는 이유와 구체적인 사례를 살펴볼 수 있습니다.

## 1. 문자열이란?

자바의 `String`은 문자의 나열을 담는 불변(immutable) 객체입니다.

### 1) 내부 구조

#### JDK 8 이하 → `char[]` 기반

#### JDK 9 이상 → Compact Strings 도입 

- 메모리 최적화된 byte[] + coder flag (문자열의 인코딩 방식)
- coder flag(0 or 1)는 문자열이 LATIN1(1바이트) 또는 UTF16(2바이트) 중 어느 방식으로 저장되는지 나타냄
- Compact Strings 란 가능한 경우 1바이트 인코딩(Latin-1)으로 저장하고, 불가능할 때만 UTF-16을 사용하는 최적화 전략
- 영어 기반 시스템에서는 문자열 메모리 사용량이 50% 가까이 감소

### 2) 불변 (Immutable)

```java
String s = "hello";
s.toUpperCase(); // 새로운 String 생성
```

- 문자열은 한 번 생성되면 절대 변경되지 않습니다.

- 이 특성 덕분에 안전하고 공유가 가능하지만, 반대로 문자열 조작이 많을수록 불필요한 객체가 계속 생성되어 성능 문제가 발생할 수 있습니다.

### 3) 문자열 상수 풀 (String Pool)

#### String Pool 이란?

```java
String a = "hello";
String b = "hello";
System.out.println(a == b);  // true
```

- JVM은 동일한 문자열 리터럴을 한 번만 생성하여 재사용합니다.

- 문자열 상수 풀이란 JVM 힙(heap) 내부의 특별한 영역에 존재하는 전역 캐시(Map 형태의 테이블)를 의미합니다.

  - JDK 7부터 String Pool은 PermGen(영구 영역) 에서 Heap 영역으로 이동했습니다.

  - 이로 인해, Pool 크기 제한이 크게 완화되고 GC 정책의 영향을 받아 Pool 관리가 훨씬 유연해졌습니다.

#### 리터럴 vs `new String()`

```java
String a = "hello";            // Pool 참조
String b = new String("hello"); // Heap에 새로 생성
```

- 리터럴은 Pool에서 공유되지만, new String()은 항상 Heap에 새로운 객체를 생성하며, Pool과는 무관합니다.

### `intern()`

```java
String c = new String("hello").intern();
System.out.println(a == c);  // true
```

- 필요하면 수동으로 intern()을 호출해 Pool에 등록할 수 있습니다.

- intern()은 다음을 수행합니다:

  - Pool에 "hello"가 존재하면 → 해당 객체의 참조 반환
  - 없으면 → 현재 문자열을 Pool에 등록 후 반환

## 2. 문자열을 사용하기에 적합하지 않은 경우

### 1) 타입을 대신하기에 적합하지 않다.

#### 문제점

```java
String age = "25";
String amount = "10000";
String latitude = "37.523";
```

- 숫자인지 문자열인지 혼동 가능
- 형식 오류를 컴파일 단계에서 잡을 수 없음
- 도메인 제약(예: 음수 금지)이 표현되지 않음

#### 대안

```java
int age = 25;
BigDecimal amount = new BigDecimal("10000");
double latitude = 37.523;
```

- 의미 있는 타입을 정확히 사용


### 2) 열거 타입을 대신하기에 적합하지 않다.

#### 문제점

```java
if ("READY".equals(status)) { ... }
```

- 오타 발생 가능
- 어떤 값이 유효한 상태인지 파악 어려움
- 유지보수 시 에러 발생 가능
- 문자열 비교 비용 비쌈

#### 대안

```java
enum Status { READY, RUNNING, STOP }

if (status == Status.READY) { ... }
```
- Enum 사용하기
  - 타입 안정성 확보 가능
  - IDE 자동완성
  - 문서화 효과

### 3) 혼합 타입을 대신하기에 적합하지 않다.

#### 문제점

```java
String key = orderId + ":" + userId;
```

- 구분자가 값 안에 포함되면 파싱 불가
- 데이터 구조가 코드에서 드러나지 않음
- 오탈자나 순서 오류가 즉시 버그로 이어짐

#### 대안

```java
record OrderUserKey(String orderId, String userId) {}
```

- 전용 타입 만들기
  - 구조가 명확해짐
  - IDE 나 컴파일러가 오류를 잡아줄 수 있음

### 4) 권한을 표현하기에 적합하지 않다.

#### 문제점

```java
ThreadLocal<String> auth = new ThreadLocal<>();
auth.set("USER_AUTH");
```

- 문자열은 ThreadLocal의 “키”가 아닙니다.
  - ThreadLocal은 ThreadLocal 객체 자체가 키이며, 문자열은 단순한 값일 뿐이라, `"USER_AUTH"`는 아무런 구분 기능을 하지 못합니다.
  - 이로 인해 ThreadLocal 인스턴스를 여러 개 만들면 어디에 어떤 값이 들어가는 파악이 어렵고, 의도치 않은 충돌이나 중복된 ThreadLocal 생성이 가능합니다.

- 문자열은 도메인 의미를 표현하지 못합니다.
  - `"USER_AUTH"`, `"ADMIN"`, `"USRE_AUTH"` (오타) 모두 허용됩니다.
  - 어떤 형태가 유효한 값인지 코드에서 드러나지 않습니다.
  - IDE의 자동완성, 컴파일 검증이 불가합니다.
  - 정책 변경 시 문자열이 흩어져 있어서 수정이 어렵습니다.

#### 대안

```java
public final class UserContext {
    private static final ThreadLocal<User> current = new ThreadLocal<>();

    public static void set(User user) { current.set(user); }
    public static User get() { return current.get(); }
    public static void clear() { current.remove(); }
}
```

```java
record AuthContext(String role) {}
ThreadLocal<AuthContext> auth = new ThreadLocal<>();
```

- 전용 타입을 사용하기
  - ThreadLocal은 보통 사용자 정보, 인증 맥락, 트랜잭션 등 하나의 의미 있는 컨텍스트 객체를 저장하는 데 사용합니다.
  - 따라서 문자열보다 전용 타입을 만드는 것이 훨씬 안전합니다.
  - 타입으로 의미가 명확해지고, 오타가 원천 차단되며, ThreadLocal 관리가 한 곳에 모여 유지보수가 쉬워집니다.

## 3. 결론

문자열은 텍스트를 표현하는 데는 적합하지만, 의미 있는 값을 표현하기에는 부적절한 타입입니다.

> 도메인 의미를 깔끔하게 표현할 수 있는 타입이 있다면, 절대 문자열로 대체하지 말도록 합시다.

- 숫자는 숫자 타입으로
- 상태는 enum으로
- 구조는 record/VO로
- 옵션은 전용 타입으로

이 규칙 하나만 지켜도 코드의 안정성, 가독성, 유지보수성은 크게 향상됩니다.

--- 

### 질문

Q. 문자열을 사용하기에 적합하지 않은 경우는 무엇이 있나요? 

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 62