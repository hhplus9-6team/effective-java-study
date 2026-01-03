# item 89. 인스턴스 수를 통제해야 한다면 readResolve보다는 열거 타입을 사용하라
> * 싱글톤패턴 사용할 때 `readResolve` 기반 방어로 불충분하니 * **enum 싱글톤** 사용하자

## 1. 문제 배경

* `Serializable`을 구현한 클래스는 역직렬화 시 **생성자를 거치지 않는다**
* 대신 JVM이:

    * 객체를 메모리에 생성하고
    * 필드를 채운 뒤
    * 필요하면 `readResolve()`를 호출한다
* 싱글톤을 보호하기 위해 `readResolve()`를 사용하는 경우가 많다

하지만
**`readResolve()`는 “최종 결과”만 바꿀 뿐,
역직렬화 도중 생성되는 객체 그래프 자체는 통제하지 못한다**

이 틈을 이용한 공격 예제가 **ElvisStealer**이다.

---

## 2. 예제에 등장하는 클래스 구조

### 2.1 Elvis (피해 클래스)
```java
public class Elvis implements Serializable {
    private static final Elvis INSTANCE = new Elvis();
    private String[] favoriteSongs = { "Hound Dog", "Heartbreak Hotel" };

    private Object readResolve() {
        return INSTANCE;
    }
}
```
* 싱글톤
* `readResolve()`로 싱글톤 보장
* 문제의 필드: `favoriteSongs`

```java
private String[] favoriteSongs;
```

* `transient`가 아님
* 참조 타입 필드 → 역직렬화 시 반드시 **객체를 따라 들어간다**

---

### 2.2 ElvisStealer (도둑 클래스)
```java
public class ElvisStealer implements Serializable {
    static Elvis impersonator;
    private Elvis payload;

    private Object readResolve() {
        impersonator = payload;
        return new String[] { "A Fool Such as I" };
    }
}
```
* `Serializable` 구현
* 두 개의 핵심 필드

```java
static Elvis impersonator; // 훔친 객체를 외부로 꺼내는 통로
private Elvis payload;     // 역직렬화 도중 채워질 필드
```

* `readResolve()`에서 payload를 훔친다

```java
private Object readResolve() {
    impersonator = payload;
    return new String[] { "A Fool Such as I" };
}
```

---

## 3. 공격의 핵심 아이디어

정상적인 객체 그래프는 다음과 같다.

```
Elvis
 └─ favoriteSongs → String[]
```

공격자가 조작한 직렬화 스트림은 다음과 같다.

```
Elvis
 └─ favoriteSongs → ElvisStealer
                       └─ payload → Elvis
```

핵심 포인트:

* `favoriteSongs` 자리에 **String[] 대신 ElvisStealer 객체**가 오도록 스트림을 조작
* ElvisStealer 내부의 `payload`가 **Elvis를 참조하도록 구성**
* Elvis ↔ ElvisStealer 간 **순환 참조** 형성

---

## 4. 역직렬화가 실제로 진행되는 순서

### 4.1 Elvis 객체 생성

* 생성자 호출 ❌
* 메모리 할당 후 **핸들(handle)** 등록
* 아직 필드는 미완성 상태

---

### 4.2 favoriteSongs 필드 처리 시작

* 참조 타입 필드이므로
* 스트림을 따라가서 값에 해당하는 객체를 역직렬화
* 스트림에는 **ElvisStealer 객체**가 있음

→ ElvisStealer 역직렬화 시작

---

### 4.3 ElvisStealer 객체 생성

* ElvisStealer 인스턴스 생성
* JVM이 핸들 등록

---

### 4.4 ElvisStealer.payload 필드 처리

* payload 타입: `Elvis`
* 스트림에 기록된 값: **이미 생성된 Elvis 객체**
* JVM은 확인:

    * “이 Elvis는 이미 핸들로 존재함”

→ **새 Elvis 객체를 만들지 않고 기존 객체 참조를 사용**

이 단계에서:

* 무한 재귀 ❌
* 순환 참조 안전하게 처리 ⭕

---

### 4.5 ElvisStealer 역직렬화 완료 → readResolve 실행

* ElvisStealer의 모든 필드가 채워진 뒤
* `ElvisStealer.readResolve()` 자동 호출

```java
impersonator = payload;
```

* 아직 `Elvis.readResolve()`는 실행되지 않음
* 즉, **미완성 Elvis 객체 참조를 훔친 상태**

---

### 4.6 readResolve 반환값 처리

* `readResolve()`가 `String[]` 반환
* JVM은 이를:

```java
Elvis.favoriteSongs = 반환된 String[];
```

로 사용

→ 타입 불일치 문제 없음

---

### 4.7 나중에 Elvis.readResolve() 실행

* 싱글톤 교체 수행
* 하지만 이미 늦음

```java
ElvisStealer.impersonator
```

에는:

* `readResolve()` 이전의 Elvis 객체 참조가 남아 있음

---

## 6. 이 예제가 말하고 싶은 결론

* `readResolve()`는 **최종 결과만 통제**할 수 있다
* 역직렬화 **중간 단계의 객체 그래프**는 통제하지 못한다
* 따라서:

    * 싱글톤
    * 불변식
    * 인스턴스 통제
      를 `readResolve()`에만 의존하면 **깨질 수 있다**

> 그냥 싱글톤 최선의 방법인 Enum을 사용하자.

---
### QnA
* 싱글톤 생성 최고의 방법은?