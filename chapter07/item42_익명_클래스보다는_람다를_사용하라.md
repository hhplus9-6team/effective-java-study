
# ITEM 42. 익명 클래스보다는 람다를 사용하라


# 익명클래스
- new 인터페이스() { ... }처럼 이름 없이 정의·인스턴스화하는 내부 클래스
- 일회성으로 구현할때 사용
```java
Collections.sort(words, new Comparator<String>() {
    @Override
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});
```


---

## 람다

### 전환된 코드
```java
Collections.sort(words, (s1, s2) -> Integer.compare(s1.length(), s2.length()));
```
- `s1`, `s2`, 반환 타입을 적지 않아도 컴파일러가 타입을 추론한다.

### 타입 추론과 제네릭 규칙
- 타입 추론은 복잡한 규칙을 따르지만, 컴파일러가 알아내지 못할 때만 타입을 명시하라
- 타입 정보를 풍부하게 제공하려면 제네릭 관련 아이템을 따르라.
  - 아이템 26: Raw 타입을 쓰지 말라.
  - 아이템 29: 제네릭을 사용하라.
  - 아이템 30: 제네릭 메서드를 사용하라.
- 컴파일러는 대부분의 타입 정보를 **제네릭**에서 얻는다.

### 비교자 생성 메서드
```java
Collections.sort(words, Comparator.comparingInt(String::length));

// 자바 8에서 List 인터페이스에 추가된 default sort 메서드 활용
words.sort(Comparator.comparingInt(String::length));

```

---

## 열거 타입

### 기존 상수별 클래스
```java
public enum Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };

    private final String symbol;

    Operation(String symbol) { this.symbol = symbol; }

    @Override public String toString() { return symbol; }

    public abstract double apply(double x, double y);
}
```

### 람다 활용
```java
public enum Operation {
    PLUS  ("+", (x, y) -> x + y),
    MINUS ("-", (x, y) -> x - y),
    TIMES ("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x / y);

    private final String symbol;
    private final DoubleBinaryOperator op;

    Operation(String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
    }

    @Override public String toString() { return symbol; }

    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}
```

---

## 람다식 사용 시 주의해야 할 사항

### 1. 람다는 이름이 없고 문서화가 불가능하다

- 람다는 이름을 붙일 수 없다.
- 코드 자체로 동작이 명확하지 않거나 줄 수가 많아지면 가독성이 급격히 떨어진다.
- 람다는 **한 줄일 때 가장 좋고**, 길어야 **세 줄 안에** 끝내야 한다.


### 2. 열거 타입 생성자에서 람다 사용 시 주의

- 열거 타입 생성자에 넘겨지는 인수들의 타입은 **컴파일타임에 추론**된다.
- 반면에 열거 타입 인스턴스는 **런타임에 만들어진다**.
- 따라서 열거 타입 생성자 안의 람다는 **열거 타입의 인스턴스 멤버에 접근할 수 없다**.

**예시:**

```java
public enum Operation {
    PLUS("+", (x, y) -> x + y),  // ✅ OK: 인스턴스 멤버 사용 안 함
    MINUS("-", (x, y) -> x - y);
    
    private final String symbol;
    private final DoubleBinaryOperator op;
    
    Operation(String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
        
        // ❌ 컴파일 에러: 람다에서 인스턴스 멤버 접근 불가
        // Operation op2 = (x, y) -> this.symbol + " 연산";  // 에러!
    }
    
    // ✅ OK: 메서드에서는 인스턴스 멤버 사용 가능
    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}
```

---

## 익명 클래스를 사용해야 하는 경우

### 1. 추상 클래스의 인스턴스를 만들 때

- 람다는 **함수형 인터페이스(단일 추상 메서드)**에서만 사용 가능하다.

### 2. 추상 메서드가 여러 개인 인터페이스의 인스턴스를 만들 때

- 람다는 단일 추상 메서드만 가진 인터페이스에서만 사용 가능하다.

### 3. 함수 객체가 자신을 참조해야 할 때

- 람다에서의 `this`는 **람다가 선언된 바깥 인스턴스**를 가리킨다.
- 익명 클래스에서의 `this`는 **익명 클래스의 인스턴스 자신**을 가리킨다.

**예시:**

```java
public class Test {
    private String name = "Test 클래스";
    
    public void testLambda() {
        Runnable r1 = () -> {
            // 람다의 this는 바깥 인스턴스(Test)를 가리킴
            System.out.println(this.getClass().getName());  // "Test"
            System.out.println(this.name);  // "Test 클래스"
        };
        r1.run();
    }
    
    public void testAnonymous() {
        Runnable r2 = new Runnable() {
            @Override
            public void run() {
                // 익명 클래스의 this는 익명 클래스 인스턴스 자신을 가리킴
                System.out.println(this.getClass().getName());  // "Test$1" (익명 클래스)
                // System.out.println(this.name);  // 컴파일 에러 (익명 클래스에 name 필드 없음)
                
                // 바깥 인스턴스에 접근하려면
                System.out.println(Test.this.name);  // "Test 클래스"
            }
        };
        r2.run();
    }
}
```
---

## 직렬화 주의점

### 람다와 익명 클래스는 직렬화하지 말라

- 람다와 익명 클래스의 직렬화 형태가 **구현별로(가상머신별로) 다를 수 있다**.

**`private static` 중첩 클래스 사용**


**왜 `private static` 중첩 클래스를 사용하나?**
- 명시적인 클래스이므로 직렬화 형태가 고정되어서 JVM이 알아서 만든 익명/람다와 달리 바뀔일이 없다

---

## 질문

### Q1. 람다식 사용 시 주의해야 할 사항 무엇인가요?

### Q2. 람다와 익명클래스는 직렬화하면 안되는 이유 무엇인가요?


