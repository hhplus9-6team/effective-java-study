# ITEM 04.인스턴스화를 막으려거든 private 생성자를 사용하라
## 주제
- 생성자를 막는 것은 클래스의 사용 목적(인스턴스화 여부)를 강제하는 수단이다
- 인스턴스화를 강제하는 경우 : util성 클래스, 싱글턴, 정적 팩터리 메서드 패턴
- 단순하게 **`interface`나 `abstract class`로 만들지 말고 `private`생성자를 사용하라**
    - 추상클래스나 인스턴스로 만든느 것은 인스턴스화를 막을 수 없으며,
      **사용자가 상속해서 사용하라는 뜻으로 오해할 수 있기 때문**이다.
```java
public class UtilityClass {
    // 인스턴스화 방지용
    private UtilityClass() {
        throw new AssertionError();
    }
}
```
- 생성자 내부에서 오류를 던지면 클래스 안에서라도 생성자를 호출하지 않도록 보장해준다.
- 그다지 직관적이지 않기에 주석 처리해주는 것을 권장한다.

> item 04는 item 01과 item 03을 보완하는 내용으로 보인다.

## 사용 예시
### 유틸리티 클래스
```java
public class UtilityClass {
    // 인스턴스화 방지를 위해 private 생성자 선언
    private UtilityClass() {
        throw new AssertionError();
    }

    public static final double PI = 3.14159;

    public static double circleArea(double r) {
        return PI * r * r;
    }
}
```
- 기본 생성자가 존재하기에 실수로라도 `new UtilityClass()`를 호출해서 인스턴스 생성 가능
- private 생성자로 인스턴스화 강제

### 싱글턴
```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {}

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```
- item 03 참고

### 정적 팩터리 강제
```java
public final class Boolean {
    public static final Boolean TRUE = new Boolean(true);
    public static final Boolean FALSE = new Boolean(false);

    private final boolean value;

    private Boolean(boolean value) { // 외부에서 new Boolean 불가
        this.value = value;
    }

    public static Boolean valueOf(boolean b) {
        return b ? TRUE : FALSE;
    }
}
```
- 클라이언트가 정적 팩터리만을 사용하여 객체를 생성하도록 강제하기 위해 private 생성자 선언
- final 클래스로 같이 선언함으로써 상속 안됨을 문서화할 수 있음.