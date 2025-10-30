# item34. int 상수 대신 열거 타입을 사용하라

## 1. int 상수의 한계

```java
// 정수 열거 패턴
public class OrderStatus {
    public static final int PENDING = 0;
    public static final int PAID = 1;
    public static final int SHIPPED = 2;
    public static final int DELIVERED = 3;
}

// 문자열 열거 패턴
public class Operation {
    public static final String PLUS = "+";
    public static final String MINUS = "-";
    public static final String TIMES = "*";
    public static final String DIVIDE = "/";
}
```

### 1) 타입 안정성 결여

```java
public class OrderStatus {
    public static final int READY = 0;
    public static final int SHIPPING = 1;
    public static final int COMPLETE = 2;
}

void updateStatus(int status) {
    // ⚠️ 잘못된 값이 들어와도 오류 없음
    if (status == OrderStatus.READY) ...
}

updateStatus(999); // 컴파일 OK → 런타임에서 논리적 오류 발생
```

- 잘못된 값이 들어가도 컴파일러가 막지 못하는 문제가 있습니다.

### 2) 가독성 저하

```java
if (status == 1) {   // 🤔 이게 뭐더라?
    System.out.println("배송 중");
}
```

- 로그에 `status=1`만 찍힌다면, 이게 PAID인지 SHIPPED인지 알 방법이 없습니다.

### 3) 확장 시 충돌 위험

```java
public class OrderStatus {
    public static final int READY = 0;
    public static final int SHIPPING = 1;
}

public class PaymentStatus {
    public static final int READY = 0;  // ⚠️ OrderStatus와 충돌
    public static final int COMPLETE = 1;
}

// ❌ 두 READY는 전혀 다른 의미인데, 값이 같음
if (orderStatus == paymentStatus) {
        System.out.println("같은 상태");
}
```

- 서로 다른 클래스에서 같은 값(예: 0)이 정의돼도 구분 불가능합니다.

### 4) 메타데이터 (부가 정보) 부족

```java
public class FeeType {
    public static final int BASIC = 0;
    public static final int PREMIUM = 1;
    public static final int VIP = 2;
}

// 요금 계산 로직은 다른 곳에 따로 둬야 함
public double calcFee(int type, double amount) {
    switch (type) {
        case FeeType.BASIC: return amount * 0.05;
        case FeeType.PREMIUM: return amount * 0.10;
        case FeeType.VIP: return amount * 0.15;
        default: throw new IllegalArgumentException("Unknown type");
    }
}
```

- 이름 이외에 부가 정보를 함께 담기가 어렵습니다.


## 2. 열거 타입 (Enum)

```java
public enum OrderStatus {
    PENDING, PAID, SHIPPED, DELIVERED;
}
```

- **열거 타입(enum type)** 은 고정된 상수 집합을 하나의 타입으로 정의하고, 그 상수들을 타입 안전한 객체로 다루게 해주는 자바의 특수한 클래스입니다.

### 1) Enum의 내부 구조

```java
public abstract class Enum<E extends Enum<E>> implements Comparable<E>, Serializable
```

- `ordinal()` : 선언된 순서를 0부터 반환
- `name()` : 상수의 이름을 반환
- `values()` : 모든 상수를 배열로 반환
- `valueOf(String name)` : 이름으로 상수를 가져옴

### 2) Enum에 메서드와 상태 추가

```java
public enum FeeType {
    BASIC(0.05),
    PREMIUM(0.10),
    VIP(0.15);

    private final double rate;

    FeeType(double rate) {
        this.rate = rate;
    }

    public double calculateFee(double amount) {
        return amount * rate;
    }
}
```

- Enum은 단순 상수 집합이 아니라 하나의 클래스이므로 필드와 메서드를 가질 수 있고, 임의의 인터페이스를 구현하게 할 수 있습니다.
  - 모든 Enum은 자동으로 java.lang.Enum<E>를 상속하며, Object 메서드들, Comparable, Serializable을 구현하고 있습니다.
  - 열거 타입은 근본적으로 불변이라 모든 필드는 final 이어야 합니다.
- 상수 하나당 자신의 인스턴스를 하나씩 만들어 `public static final` 필드로 공개합니다.
- 열거 타입은 밖에서 접근할 수 있는 생성자를 제공하지 않으므로 사실상 final 입니다.
- 열거 타입은 열거 타입의 값끼리 == 연산자로 비교하기 때문에 타입 안정성을 제공합니다.

### 3) 상수별 메서드 구현

```java
public enum Operation {
    PLUS, MINUS, MULTIPLY, DIVIDE;

    public double apply(double x, double y) {
        switch (this) {
            case PLUS:      return x + y;
            case MINUS:     return x - y;
            case MULTIPLY:  return x * y;
            case DIVIDE:    return x / y;
            default:
                throw new AssertionError("Unknown operation: " + this);
        }
    }
}
```

- switch 문을 사용하는 경우 새로운 상수를 추가하면 해당 case 문을 추가해줘야 합니다.
- 각 상수의 로직이 한 곳에 몰려 있어 응집도가 낮고, 코드가 길어지며 가독성이 떨어집니다.
- 그래도 switch문은 상수별 동작을 혼합해 넣을 때 좋은 선택이 될 수 있습니다.
    ```java
    public static Operation inverse(Operation op) {
        switch(op) {
            case PLUS:   return Operation.MINUS;
            case MINUS:  return Operation.PLUS;
            case TIMES:  return Operation.DIVIDE;
            case DIVIDE: return Opearation.TIMES;
            default: throw new AssertionError("알 수 없는 연산");
        }
    }
    ```

```java
public enum Operation {
    PLUS {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS {
        public double apply(double x, double y) { return x - y; }
    },
    MULTIPLY {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE {
        public double apply(double x, double y) { return x / y; }
    };

    public abstract double apply(double x, double y);
}
```

- 열거 타입에서는 상수별로 다르게 동작하는 코드를 구현하는 더 나은 수단인 **상수별 메서드 구현** 을 제공합니다.
- 각 열거 상수가 자신만의 메서드 구현을 가집니다.
- 전략 패턴과 동일한 구조로, Enum 상수마다 다른 전략을 내장한 것과 같습니다.
- 상수별 메서드 구현을 상수별 데이터와 결합할 수도 있습니다.

```java
interface PayStrategy {
    int calculate(int base);
}

public enum PayType {
    HOURLY(base -> base * 1),          // 시급제
    SALARY(base -> base * 12),         // 연봉제
    FREELANCER(base -> base / 2);      // 프리랜서

    private final PayStrategy strategy;

    PayType(PayStrategy strategy) {
        this.strategy = strategy;
    }

    public int pay(int base) {
        return strategy.calculate(base);
    }
}
```

- 열거 상수마다 전략을 직접 가지는 대신, 공통 인터페이스를 외부로 분리하여 **전략 열거 타입 패턴** 을 사용할 수도 있습니다.
- 각 상수가 전략을 직접 갖거나, 전략 인터페이스를 위임받아 동작을 분리할 수 있습니다.


### 4) 열거 타입용 fromString 메서드 구현

```java
public enum Color {
    RED, BLUE;
}

System.out.println(Color.RED.toString());      // "RED"
System.out.println(Color.valueOf("RED"));     // Color.RED
```
- 기본적으로 모든 enum은 toString()과 valueOf(String) 메서드를 제공합니다.

```java
public enum Color {
    RED("빨강"), BLUE("파랑");

    private final String label;

    Color(String label) { this.label = label; }

    @Override
    public String toString() {
        return label;
    }
}

System.out.println(Color.RED); // 출력: 빨강
Color c = Color.valueOf("빨강"); // ❌ IllegalArgumentException
```

- valueOf()는 enum 상수 이름("RED", "BLUE") 으로만 찾을 수 있습니다.
- toString()을 "빨강"처럼 바꿔버리면, toString() ↔ valueOf()가 더 이상 짝이 안 맞게 됩니다.

```java
public enum Color {
    RED("빨강"), BLUE("파랑");

    private final String label;

    Color(String label) { this.label = label; }

    @Override
    public String toString() {
        return label;
    }

    private static final Map<String, Color> LABEL_TO_COLOR = new HashMap<>();

    static {
        for (Color c : values()) {
            LABEL_TO_COLOR.put(c.toString(), c);
        }
    }

    public static Optional<Color> fromString(String label) {
        return Optional.ofNullable(LABEL_TO_COLOR.get(label));
    }
}
```

- 열거 타입의 toString 메서드를 재정의하려거든, toString이 반환하는 문자열을 해당 열거 타입 상수로 변환해주는 fromString 메서드도 함께 제공하는 걸 고려해봅시다.
- 이를 통해 다시 양변환 변환이 가능해집니다.

### 5) 열거 타입의 초기화

```java
public enum Color {
    RED, BLUE;

    private static int counter = init(); // 상수 아님

    Color() {
        System.out.println(counter);     // ❌ enum 생성자에서 상수 아닌 static 참조 → 컴파일 에러
    }

    private static int init() { return 42; }
}
```

- enum 생성자에서 “상수(compile-time constant)가 아닌 정적 필드”를 참조하면 컴파일 자체가 실패합니다.

```java
public enum Color {
    RED, BLUE;

    private static final int CONST = 1; // 컴파일타임 상수

    Color() {
        System.out.println(CONST);      // ✅ 가능
    }
}
```

- 생성자에서 허용되는 건 “컴파일타임 상수”뿐입니다.

```java
public enum Color {
    RED, BLUE;

    private static int counter = init(); // OK: 필드 자체는 컴파일 됨

    // enum 상수들이 만들어진 '뒤'에 실행됨
    static {
        System.out.println("counter after init = " + counter); // ✅ 가능
    }

    Color() {
        System.out.println("ctor: hi"); // 여기선 counter 같은 상수 아닌 static 금지
    }

    private static int init() { return 42; }
}
```

- 생성자에서 정적 가변/비상수를 쓰지 말고, ‘나중’에 참조하여야 합니다.
- 생성자 대신 정적 초기화 블록(상수 생성 후, enum 상수들이 모두 만들어진 뒤 실행됨)이나 인스턴스 메서드에서 사용해야 합니다.

```java
public enum Color {
    RED, BLUE;

    Color() {
        // 다른 타입의 정적 멤버를 통해 지연 초기화 (허용)
        System.out.println(CounterHolder.COUNTER); // ✅ 컴파일/실행 OK
    }

    private static class CounterHolder {
        static final int COUNTER = init(); // 비상수지만 '다른 클래스'의 정적 초기화
        static int init() { return 42; }
    }
}
```

- 다른 클래스의 static 멤버를 생성자에서 참조하는 건 허용됩니다(그 타입이 그때 초기화됨).
- 따라서, 다른 타입의 홀더를 통해 ‘지연 초기화’해서 생성자에서 쓰는 것은 가능합니다.


### 5) EnumSet / EnumMap

```java
EnumSet<OrderStatus> pending = EnumSet.of(OrderStatus.READY, OrderStatus.SHIPPING);
EnumMap<OrderStatus, String> messages = new EnumMap<>(OrderStatus.class);
messages.put(OrderStatus.READY, "상품 준비 중");
```
# item34. int 상수 대신 열거 타입을 사용하라

## 1. int 상수의 한계

```java
// 정수 열거 패턴
public class OrderStatus {
    public static final int PENDING = 0;
    public static final int PAID = 1;
    public static final int SHIPPED = 2;
    public static final int DELIVERED = 3;
}

// 문자열 열거 패턴
public class Operation {
    public static final String PLUS = "+";
    public static final String MINUS = "-";
    public static final String TIMES = "*";
    public static final String DIVIDE = "/";
}
```

### 1) 타입 안정성 결여

```java
public class OrderStatus {
    public static final int READY = 0;
    public static final int SHIPPING = 1;
    public static final int COMPLETE = 2;
}

void updateStatus(int status) {
    // ⚠️ 잘못된 값이 들어와도 오류 없음
    if (status == OrderStatus.READY) ...
}

updateStatus(999); // 컴파일 OK → 런타임에서 논리적 오류 발생
```

- 잘못된 값이 들어가도 컴파일러가 막지 못하는 문제가 있습니다.

### 2) 가독성 저하

```java
if (status == 1) {   // 🤔 이게 뭐더라?
    System.out.println("배송 중");
}
```

- 로그에 `status=1`만 찍힌다면, 이게 PAID인지 SHIPPED인지 알 방법이 없습니다.

### 3) 확장 시 충돌 위험

```java
public class OrderStatus {
    public static final int READY = 0;
    public static final int SHIPPING = 1;
}

public class PaymentStatus {
    public static final int READY = 0;  // ⚠️ OrderStatus와 충돌
    public static final int COMPLETE = 1;
}

// ❌ 두 READY는 전혀 다른 의미인데, 값이 같음
if (orderStatus == paymentStatus) {
        System.out.println("같은 상태");
}
```

- 서로 다른 클래스에서 같은 값(예: 0)이 정의돼도 구분 불가능합니다.

### 4) 메타데이터 (부가 정보) 부족

```java
public class FeeType {
    public static final int BASIC = 0;
    public static final int PREMIUM = 1;
    public static final int VIP = 2;
}

// 요금 계산 로직은 다른 곳에 따로 둬야 함
public double calcFee(int type, double amount) {
    switch (type) {
        case FeeType.BASIC: return amount * 0.05;
        case FeeType.PREMIUM: return amount * 0.10;
        case FeeType.VIP: return amount * 0.15;
        default: throw new IllegalArgumentException("Unknown type");
    }
}
```

- 이름 이외에 부가 정보를 함께 담기가 어렵습니다.


## 2. 열거 타입 (Enum)

```java
public enum OrderStatus {
    PENDING, PAID, SHIPPED, DELIVERED;
}
```

- **열거 타입(enum type)** 은 고정된 상수 집합을 하나의 타입으로 정의하고, 그 상수들을 타입 안전한 객체로 다루게 해주는 자바의 특수한 클래스입니다.

### 1) Enum의 내부 구조

```java
public abstract class Enum<E extends Enum<E>> implements Comparable<E>, Serializable
```

- `ordinal()` : 선언된 순서를 0부터 반환
- `name()` : 상수의 이름을 반환
- `values()` : 모든 상수를 배열로 반환
- `valueOf(String name)` : 이름으로 상수를 가져옴

### 2) Enum에 메서드와 상태 추가

```java
public enum FeeType {
    BASIC(0.05),
    PREMIUM(0.10),
    VIP(0.15);

    private final double rate;

    FeeType(double rate) {
        this.rate = rate;
    }

    public double calculateFee(double amount) {
        return amount * rate;
    }
}
```

- Enum은 단순 상수 집합이 아니라 하나의 클래스이므로 필드와 메서드를 가질 수 있고, 임의의 인터페이스를 구현하게 할 수 있습니다.
    - 모든 Enum은 자동으로 java.lang.Enum<E>를 상속하며, Object 메서드들, Comparable, Serializable을 구현하고 있습니다.
    - 열거 타입은 근본적으로 불변이라 모든 필드는 final 이어야 합니다.
- 상수 하나당 자신의 인스턴스를 하나씩 만들어 `public static final` 필드로 공개합니다.
- 열거 타입은 밖에서 접근할 수 있는 생성자를 제공하지 않으므로 사실상 final 입니다.
- 열거 타입은 열거 타입의 값끼리 == 연산자로 비교하기 때문에 타입 안정성을 제공합니다.

### 3) 상수별 메서드 구현

```java
public enum Operation {
    PLUS, MINUS, MULTIPLY, DIVIDE;

    public double apply(double x, double y) {
        switch (this) {
            case PLUS:      return x + y;
            case MINUS:     return x - y;
            case MULTIPLY:  return x * y;
            case DIVIDE:    return x / y;
            default:
                throw new AssertionError("Unknown operation: " + this);
        }
    }
}
```

- switch 문을 사용하는 경우 새로운 상수를 추가하면 해당 case 문을 추가해줘야 합니다.
- 각 상수의 로직이 한 곳에 몰려 있어 응집도가 낮고, 코드가 길어지며 가독성이 떨어집니다.
- 그래도 switch문은 상수별 동작을 혼합해 넣을 때 좋은 선택이 될 수 있습니다.
    ```java
    public static Operation inverse(Operation op) {
        switch(op) {
            case PLUS:   return Operation.MINUS;
            case MINUS:  return Operation.PLUS;
            case TIMES:  return Operation.DIVIDE;
            case DIVIDE: return Opearation.TIMES;
            default: throw new AssertionError("알 수 없는 연산");
        }
    }
    ```

```java
public enum Operation {
    PLUS {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS {
        public double apply(double x, double y) { return x - y; }
    },
    MULTIPLY {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE {
        public double apply(double x, double y) { return x / y; }
    };

    public abstract double apply(double x, double y);
}
```

- 열거 타입에서는 상수별로 다르게 동작하는 코드를 구현하는 더 나은 수단인 **상수별 메서드 구현** 을 제공합니다.
- 각 열거 상수가 자신만의 메서드 구현을 가집니다.
- 전략 패턴과 동일한 구조로, Enum 상수마다 다른 전략을 내장한 것과 같습니다.
- 상수별 메서드 구현을 상수별 데이터와 결합할 수도 있습니다.

```java
interface PayStrategy {
    int calculate(int base);
}

public enum PayType {
    HOURLY(base -> base * 1),          // 시급제
    SALARY(base -> base * 12),         // 연봉제
    FREELANCER(base -> base / 2);      // 프리랜서

    private final PayStrategy strategy;

    PayType(PayStrategy strategy) {
        this.strategy = strategy;
    }

    public int pay(int base) {
        return strategy.calculate(base);
    }
}
```

- 열거 상수마다 전략을 직접 가지는 대신, 공통 인터페이스를 외부로 분리하여 **전략 열거 타입 패턴** 을 사용할 수도 있습니다.
- 각 상수가 전략을 직접 갖거나, 전략 인터페이스를 위임받아 동작을 분리할 수 있습니다.


### 4) 열거 타입용 fromString 메서드 구현

```java
public enum Color {
    RED, BLUE;
}

System.out.println(Color.RED.toString());      // "RED"
System.out.println(Color.valueOf("RED"));     // Color.RED
```
- 기본적으로 모든 enum은 toString()과 valueOf(String) 메서드를 제공합니다.

```java
public enum Color {
    RED("빨강"), BLUE("파랑");

    private final String label;

    Color(String label) { this.label = label; }

    @Override
    public String toString() {
        return label;
    }
}

System.out.println(Color.RED); // 출력: 빨강
Color c = Color.valueOf("빨강"); // ❌ IllegalArgumentException
```

- valueOf()는 enum 상수 이름("RED", "BLUE") 으로만 찾을 수 있습니다.
- toString()을 "빨강"처럼 바꿔버리면, toString() ↔ valueOf()가 더 이상 짝이 안 맞게 됩니다.

```java
public enum Color {
    RED("빨강"), BLUE("파랑");

    private final String label;

    Color(String label) { this.label = label; }

    @Override
    public String toString() {
        return label;
    }

    private static final Map<String, Color> LABEL_TO_COLOR = new HashMap<>();

    static {
        for (Color c : values()) {
            LABEL_TO_COLOR.put(c.toString(), c);
        }
    }

    public static Optional<Color> fromString(String label) {
        return Optional.ofNullable(LABEL_TO_COLOR.get(label));
    }
}
```

- 열거 타입의 toString 메서드를 재정의하려거든, toString이 반환하는 문자열을 해당 열거 타입 상수로 변환해주는 fromString 메서드도 함께 제공하는 걸 고려해봅시다.
- 이를 통해 다시 양변환 변환이 가능해집니다.

### 5) 열거 타입의 초기화

```java
public enum Color {
    RED, BLUE;

    private static int counter = init(); // 상수 아님

    Color() {
        System.out.println(counter);     // ❌ enum 생성자에서 상수 아닌 static 참조 → 컴파일 에러
    }

    private static int init() { return 42; }
}
```

- enum 생성자에서 “상수(compile-time constant)가 아닌 정적 필드”를 참조하면 컴파일 자체가 실패합니다.

```java
public enum Color {
    RED, BLUE;

    private static final int CONST = 1; // 컴파일타임 상수

    Color() {
        System.out.println(CONST);      // ✅ 가능
    }
}
```

- 생성자에서 허용되는 건 “컴파일타임 상수”뿐입니다.

```java
public enum Color {
    RED, BLUE;

    private static int counter = init(); // OK: 필드 자체는 컴파일 됨

    // enum 상수들이 만들어진 '뒤'에 실행됨
    static {
        System.out.println("counter after init = " + counter); // ✅ 가능
    }

    Color() {
        System.out.println("ctor: hi"); // 여기선 counter 같은 상수 아닌 static 금지
    }

    private static int init() { return 42; }
}
```

- 생성자에서 정적 가변/비상수를 쓰지 말고, ‘나중’에 참조하여야 합니다.
- 생성자 대신 정적 초기화 블록(상수 생성 후, enum 상수들이 모두 만들어진 뒤 실행됨)이나 인스턴스 메서드에서 사용해야 합니다.

```java
public enum Color {
    RED, BLUE;

    Color() {
        // 다른 타입의 정적 멤버를 통해 지연 초기화 (허용)
        System.out.println(CounterHolder.COUNTER); // ✅ 컴파일/실행 OK
    }

    private static class CounterHolder {
        static final int COUNTER = init(); // 비상수지만 '다른 클래스'의 정적 초기화
        static int init() { return 42; }
    }
}
```

- 다른 클래스의 static 멤버를 생성자에서 참조하는 건 허용됩니다(그 타입이 그때 초기화됨).
- 따라서, 다른 타입의 홀더를 통해 ‘지연 초기화’해서 생성자에서 쓰는 것은 가능합니다.


### 6) EnumSet / EnumMap

```java
EnumSet<OrderStatus> pending = EnumSet.of(OrderStatus.READY, OrderStatus.SHIPPING);
EnumMap<OrderStatus, String> messages = new EnumMap<>(OrderStatus.class);
messages.put(OrderStatus.READY, "상품 준비 중");
```

- 열거 타입은 고정된 상수 집합이기 때문에, HashSet이나 HashMap처럼 범용 컬렉션보다 훨씬 효율적인 전용 구현체(EnumSet, EnumMap)를 사용할 수 있습니다.
  - EnumSet은 내부적으로 비트 벡터 기반이라 HashSet보다 훨씬 빠릅니다. 
  - EnumMap은 배열 기반이라 HashMap보다 가볍습니다.

## 3. 결론

> - 필요한 원소를 컴파일타임에 다 알 수 있는 상수 집합이라면 항상 열거 타입을 사용합시다.
> - 열거 타입에 정의된 상수 개수가 영원히 고정 불변일 필요는 없습니다.