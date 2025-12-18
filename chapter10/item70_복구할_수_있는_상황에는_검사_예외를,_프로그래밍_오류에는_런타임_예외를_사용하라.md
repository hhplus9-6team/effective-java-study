# item70. 복구할 수 있는 상황에는 검사 예외를, 프로그래밍 오류에는 런타임 예외를 사용하라

## 1. 서론

자바에서 예외 설계는 단순한 기술 요소가 아니라 API 품질을 결정하는 핵심 설계 원칙입니다.

예외를 잘못 사용하면 클라이언트 코드를 복잡하게 만들고, 오류를 숨기거나 개발자가 의도하지 않은 방향으로 코드를 작성하게 만듭니다.

이 아이템에서는 예외의 종류와 언제 어떤 예외를 선택해야 하는지를 확인할 수 있습니다.

## 2. 자바의 예외

```
java.lang.Object
   └── java.lang.Throwable   ← 예외 최상위 클래스
         ├── java.lang.Exception     ← 애플리케이션 레벨 예외
         │      ├── Checked Exception     ← 반드시 처리해야 함
         │      └── RuntimeException      ← 프로그래밍 오류(Unchecked)
         └── java.lang.Error         ← JVM 시스템 오류
```

자바의 모든 예외는 `Throwable`을 최상위로 하는 계층 구조를 가집니다.

### 1) Throwable

- catch 가능한 모든 예외/오류의 슈퍼 타입
- 두 가지 하위 분류만 존재함 : `Exception`, `Error`

### 2) Exception

- 애플리케이션 로직에서 발생하는 문제
- 자바는 “복구 가능한 문제”와 “개발자가 고쳐야 할 문제”를 명확히 구분하기 위해 Checked/Unchecked 방식을 도입
- Checked는 호출자가 대응해야 하고, Unchecked는 개발자가 고쳐야 함

### 3) Checked Exception - 복구할 수 있는 상황

- 컴파일러가 꼭 처리하도록 강제하는 예외
- `try-catch` 또는 `throws`가 필수
- API 설계자가 "호출하는 쪽에서 이 상황을 처리해야 한다"고 메시지를 주는 것
- 예 : `IOException`, `SQLException`, `ParseException`

#### 복구할 수 있는 상황이란?

> 복구 가능하다는 것은 '개발자의 잘못이 아니라 외부 환경의 문제이므로 호출자가 적절한 조치를 취하면 프로그램을 계속 진행할 수 있다'는 의미.

(1) 파일을 읽으려고 했는데 없을 때

```java
try {
    readConfig();
} catch (FileNotFoundException e) {
    // 대응 가능
    createDefaultConfig();
}
```
- 파일을 새로 만들거나 다른 경로에서 다시 읽으면 복구 가능 

(2) 네트워크 연결이 끊겼을 때

```java
try {
    client.sendRequest();
} catch (IOException e) {
    retry();
}
```
- 재시도하여 성공하면 복구 가능

(3) 사용자가 잘못된 입력을 넣었을 때

```java
try {
    parseUserAge(input);
} catch (ParseException e) {
    askAgain();  
}
```

- 다시 입력받으면 복구 가능

### 4) RuntimeException - 프로그래밍 오류 (Unchecked)

- 컴파일러가 처리 강제를 하지 않음
- 보통 코드 자체의 결함을 의미
- 복구할 수 없고, 실패해야 함
- 예 : `NullPointerException`, `IllegalArgumentException`, `IndexOutOfBoundsException`, `IllegalStateException`

#### 프로그래밍 오류란?

> - 호출자가 아무리 try-catch로 잡아도 복구가 불가능하다. 
> - 오히려 예외를 조용히 삼켜버리면 디버깅이 훨씬 어려워진다.
> - 그래서 "빠르게 실패"하여 개발자가 고치도록 유도하는 것이 맞다.

(1) 잘못된 인자를 넘김 → IllegalArgumentException

```java
setAge(-1); // IllegalArgumentException
```

(2) 객체 상태가 잘못됨 → IllegalStateException

```java
iterator.remove(); // next() 이전에 호출 → IllegalStateException
```

(3) null 사용 → NullPointerException

```java
obj.toString();  // NullPointerException
```
- 아예 로직 오류

(4) 배열 인덱스 범위 초과 → IndexOutOfBoundsException

```java
arr[10];  // IndexOutOfBoundsException
```
- 다시 시도한다고 해결되는 문제가 아님

### 5) Error

- JVM 내부의 심각한 문제
- 애플리케이션이 복구할 수 없음
- Error는 JVM 상태가 이미 손상된 상태이기 때문에 catch해서 회복을 시도하는 것은 오히려 더 위험
- 일반 애플리케이션에서 Error를 던지도록 설계하면 안됨
- 예 : `OutOfMemoryError`, `StackOverflowError`, `LinkageError`

## 3. 결론

> ✔ 복구 가능한 외부 요인 → Checked Exception 
>
> ✔ 프로그래밍 오류 → RuntimeException
> 
> ✔ 시스템 레벨 오류 → Error

분류 | 의미 | 사용 목적
-|-|-
Checked | 개발자가 반드시 처리해야 하는 외부 문제 | 클라이언트에게 "이 상황에 대응하라"는 신호
Uncheckd(Runtime) | 코드 오류 또는 API 오용 | 빠른 실패
Error | JVM 레벨 에러 | 개발자가 건드리는 영역 아님


---

### 질문

Q. 다음 4가지 상황에서 각각 어떤 예외가 적절한가요? (CheckedException, RuntimeException, Error 중에 선택)

A) 외부 시스템이 일시적으로 다운됨 → ? 

B) 메서드 호출 순서가 잘못됨 → ? 

C) 배열 인덱스가 잘못됨 → ?

D) JVM 메모리가 부족함 → ?

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 70
