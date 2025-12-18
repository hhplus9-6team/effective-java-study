# ITEM 63. 문자열 연결은 느리니 주의하라

**문자열 연결 연산자(`+`)는 여러 문자열을 연결할 때 사용하면 성능이 크게 저하된다. 반복문 안에서 문자열 연결이 필요하면 `StringBuilder`를 사용하라.**

## String '+' 연산의 내부 동작

### 컴파일 타임 변환

Java에서 String `+` 연산자는 Java 컴파일러에서 구현되며, 컴파일 타임에 내부적으로 `StringBuilder` 클래스를 만든 후 다시 문자열로 반환합니다.

```java
String banana = "바나나";
banana += "쥬스";

// 위 코드는 컴파일 타임에 아래와 같이 변환됨
String banana = new StringBuilder("바나나").append("쥬스").toString();
```

### 반복문에서의 문제점

```java
public class Test {
    public static void main(String[] args) {
        String number = "0";
        for (int i = 1; i <= 10000; i++) {
            number += "i";  // 매번 StringBuilder 객체 생성
        }
    }
}
```

**컴파일 후 실제 동작:**
```java
String number = "0";
for (int i = 1; i <= 10000; i++) {
    number = new StringBuilder(number).append(i).toString();
    // 반복문의 횟수만큼 StringBuilder 객체가 생성되고
    // append()와 toString() 메서드 호출이 매번 발생
}
```

**문제점:**
- 반복문의 횟수만큼 `StringBuilder` 객체가 생성됨
- `append()`와 `toString()` 메서드 호출이 매번 발생
- 성능이 저하되고 메모리 낭비가 커짐

**결론**
```java
public class Test {
    public static void main(String[] args) {
        StringBuilder number = new StringBuilder("0");
        for (int i = 1; i <= 10000; i++) {
            number.append(i);
        }
        String result = number.toString();
    }
}
```

**장점:**
- `StringBuilder` 객체를 한 번만 생성
- 반복문 내에서는 `append()`만 호출
- 성능이 크게 향상됨

---

## 질문

### Q1. 반복문 안에서 String '+' 연산을 사용하면 왜 성능이 저하되나요?

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 63
