# item90. 직렬화된 인스턴스 대신 직렬화 프록시 사용을 검토하라

> [!IMPORTANT]
> `Serializable`을 구현하는 순간, **생성자를 거치지 않고도 인스턴스를 만들 수 있게 된다** -> **버그/보안 문제 가능성**
>
> 이 위험을 크게 줄이는 대표 기법이 **직렬화 프록시 패턴**이다


## 왜 위험해지나?
`Serializable`을 구현하면 역직렬화가 **생성자를 우회**해 객체를 만들 수 있다
즉, 평소라면 생성자/정적 팩터리에서 강제하던 **불변식 검증, 방어적 복사** 같은 안전장치를 우회할 여지가 생긴다

<details>
<summary>예제: 생성자 우회로 불변식이 깨지는 경우</summary>

```java
public class Age implements Serializable {
    private final int value;

    public Age(int value) {
        if (value < 0) throw new IllegalArgumentException("나이는 음수일 수 없습니다");
        this.value = value;
    }
}
```
- 생성자는 `value < 0`이면 예외를 던져 **불변식을 보장**한다
- 하지만 역직렬화는 생성자를 거치지 않으므로, 조작된 바이트 스트림으로 `value = -100`인 객체가 만들어질 수 있다

</details>

** 직렬화를 해야만 한다면, **직렬화가 객체를 '직접' 만들게 두지 말고 프록시(대리 객체)를 통해서만** 복원되게 만들라


### 자바 직렬화 특수 메서드
```java
// 자바가 인식하는 특수 메서드들 (있으면 자동 호출)
private void writeObject(ObjectOutputStream s)     // 직렬화 시
private void readObject(ObjectInputStream s)       // 역직렬화 시
private Object writeReplace()                      // 직렬화 전
private Object readResolve()                       // 역직렬화 후
```

## 직렬화 프록시 패턴(serialization proxy pattern)이란?
바깥(원래) 클래스의 **논리적 상태(logical state)** 를 정확히 표현하는 **중첩 클래스**를 하나 만들고,
그 중첩 클래스를 **직렬화의 실제 대상**으로 쓰는 방식이다.

### 1) `private static` 중첩 프록시 클래스
- 바깥 클래스의 논리적 상태를 담는 필드들로만 구성
- **생성자는 딱 1개**
  - 바깥 클래스 인스턴스를 받아서
  - 그 안의 데이터를 **그대로 복사**한다
- 설계상 프록시의 직렬화 형태는 바깥 클래스의 직렬화 형태로 쓰기에 “이상적”이다.
- 이 생성자에서는 **일관성 검사/방어적 복사도 필요 없다**(프록시가 곧 논리 상태의 스냅샷이기 때문).

#### 예: `Period`용 직렬화 프록시
```java
private static class SerializationProxy implements Serializable {
    private final Date start;
    private final Date end;

    SerializationProxy(Period p) {
        this.start = p.start;
        this.end = p.end;
    }

    private static final long serialVersionUID = 234098243823485285L; // 아무 값이나 상관없다(아이템 87)
}
```

### 2) 바깥 클래스에 `writeReplace` 추가
바깥 클래스의 인스턴스가 직렬화되려 할 때,
자바 직렬화 시스템이 **바깥 클래스 대신 프록시 인스턴스를 반환**하게 만든다.

```java
// 직렬화 프록시 패턴용 writeReplace 메서드
private Object writeReplace() {
    return new SerializationProxy(this);
}
```
이 메서드 덕분에 직렬화 시스템은 **결코 바깥 클래스의 ‘직렬화된 인스턴스’를 만들어낼 수 없다.**


### 3) 바깥 클래스에 `readObject`를 추가
공격자는 ‘가짜 바이트 스트림’으로 불변식을 깨는 역직렬화를 시도할 수 있다.
바깥 클래스 쪽 `readObject`를 아래처럼 만들어 두면, **프록시 없이 바깥 클래스를 직접 역직렬화하는 시도**를 막을 수 있다.

```java
// 직렬화 프록시 패턴용 readObject 메서드
private void readObject(ObjectInputStream stream) throws InvalidObjectException {
    throw new InvalidObjectException("프록시가 필요합니다.");
}
```

## 프록시에서 실제 객체로 복원: `readResolve`
프록시 클래스에 `readResolve`를 추가한다.
이 메서드는 역직렬화 시 프록시를 **바깥 클래스와 논리적으로 동일한 인스턴스**로 바꿔치기 한다.

핵심은:
- `readResolve`는 **공개된 API(생성자/정적 팩터리 등)** 를 사용해 바깥 클래스 인스턴스를 만든다.
- 그 결과 직렬화가 제공하는 “생성자 우회” 특성이 **상당 부분 제거**된다.

#### `Period` 프록시의 `readResolve`
```java
private Object readResolve() {
    return new Period(start, end); // public 생성자를 사용한다.
}
```

이 방식이 아름다운 이유:
- 역직렬화된 인스턴스가 불변식을 만족하는지 검사하거나,
- 추가적인 방어 수단을 강구할 필요가 크게 줄어든다.
- 바깥 클래스의 생성자/정적 팩터리/인스턴스 메서드가 평소처럼 불변식을 지켜주면, 더 해줘야 할 일이 거의 없다.


## 직렬화 프록시의 장점

### 1) 특정 직렬화 공격을 프록시 수준에서 차단
방어적 복사(예: 아이템 88의 `readObject` 방어적 복사)보다,
직렬화 프록시는 다음 공격을 **프록시 단계에서** 막아준다고 설명한다.
- **가짜 바이트 스트림 공격**
- **내부 필드 탈취 공격**

### 2) 불변(immutable) 설계를 더 쉽게 만든다
프록시 패턴을 쓰면 `Period`의 필드를 `final`로 선언해
클래스를 진정한 불변(아이템 17)으로 만드는 것도 수월해진다.

### 3) 역직렬화 결과 클래스가 달라도 정상 동작
직렬화 프록시는 “논리 상태”만 담고,
복원은 공개 API로 하므로
**원래 직렬화된 인스턴스의 클래스와 역직렬화로 만들어지는 클래스가 달라도** 잘 동작한다.

#### 예: `EnumSet`
- `EnumSet`은 `public` 생성자 없이 **정적 팩터리만 제공**한다.
- 현재 OpenJDK 구현은 열거 타입 원소 수에 따라
  - 원소가 64개 이하면 `RegularEnumSet`
  - 더 크면 `JumboEnumSet`
  중 하나를 사용한다.

이때 어떤 시점에 직렬화된 구현 클래스가 나중에는 최선이 아닐 수 있는데,
직렬화 프록시를 쓰면 역직렬화 시점에 **현재 최선의 구현**을 선택해 만들 수 있다.
(`elementType`과 `elements`를 저장해 두었다가, `readResolve`에서 `EnumSet.noneOf(elementType)`로 만들고 요소를 다시 추가한다.)

## 한계

1) **클라이언트가 멋대로 확장할 수 있는 클래스(아이템 19)** 에는 적용할 수 없다.

2) **객체 그래프에 순환(cycle)이 있는 클래스**에도 적용할 수 없다.
- 이런 객체의 메서드를 프록시의 `readResolve` 안에서 호출하려 하면 `ClassCastException`이 발생할 수 있다
- 이유: 그 시점에는 프록시만 존재하고, 실제 객체는 아직 만들어진 게 아니기 때문.

## 성능 트레이드오프
직렬화 프록시 패턴은 강력하고 안전하지만 비용이 있다.
`Period` 예제를 실행해보면 **방어적 복사보다 14% 느렸다**고 언급한다.

## 핵심 정리
- 제3자가 확장할 수 없는 클래스라면, 가능한 한 **직렬화 프록시 패턴**을 사용하자.
- 이 패턴은 중요한 불변식을 **안정적으로 직렬화**해주는 가장 쉬운 방법일 수 있다.

## QnA
Q1. 바깥 클래스에 `readObject`를 예외로 막아야 하는 이유는?  
Q2. 이 패턴을 적용할 수 없는 두 가지 경우(책에서 언급)는?
