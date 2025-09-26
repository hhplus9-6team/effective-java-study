# 싱글턴 패턴 구현 방법

## 싱글턴이란?
인스턴스를 오직 하나만 생성할 수 있는 클래스를 말합니다. 함수와 같은 무상태 객체나 설계상 유일해야 하는 시스템 컴포넌트가 대표적인 예입니다.

## 싱글턴 구현의 3가지 방법

### 1. public static final 필드 방식

```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() {...}
    public void leaveTheBuilding() {...}
}
```

**장점:**
- API에 싱글턴임이 명백히 드러남
- 간결함

**주의사항:** 리플렉션 공격 가능 (AccessibleObject.setAccessible 사용)

### 2. 정적 팩토리 메서드 방식

```java
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() {...}
    public static Elvis getInstance() { return INSTANCE; }
    public void leaveTheBuilding() {...}
}
```

**장점:**
- API 변경 없이 싱글턴이 아니게 변경 가능
- 제네릭 싱글턴 팩토리로 만들 수 있음
- 메서드 참조를 공급자로 사용 가능 (`Elvis::getInstance`)

### 3. 열거 타입 방식 (권장)

```java
public enum Elvis {
    INSTANCE;
    public void leaveTheBuilding() {...}
}
```

**장점:**
- 가장 간결함
- 직렬화 자동 처리
- 리플렉션 공격에 완벽히 안전함

**제약사항:** Enum 외의 클래스 상속 불가

## 직렬화 시 주의사항

싱글턴 클래스를 직렬화할 때는 `readResolve` 메서드를 추가해야 합니다:

```java
private Object readResolve() {
    return INSTANCE;
}
```

## 스터디 확인 문제

**다음 중 싱글턴 패턴의 세 가지 구현 방식에 대한 설명으로 올바르지 않은 것은?**

1. public static final 필드 방식은 API에 싱글턴임이 명백히 드러나는 장점이 있다.

2. 정적 팩토리 메서드 방식은 API 변경 없이 싱글턴이 아니게 변경할 수 있는 유연성을 제공한다.

3. 열거 타입 방식은 직렬화와 리플렉션 공격에 대해 완벽한 보호를 제공한다.

4. 열거 타입 방식은 다른 클래스를 상속받아야 하는 싱글턴에서도 사용할 수 있다.

5. 싱글턴 클래스를 직렬화할 때는 readResolve 메서드를 구현해야 새로운 인스턴스 생성을 방지할 수 있다.
