# item 38. 확장할 수 있는 열거 타입을 인터페이스로 정의하라
> * 자바의 `enum`은 **언어 차원에서 확장 불가능**하지만, 
>   **인터페이스를 이용하면 확장 가능성을 “모방”할 수 있다.**
> * `<T extends Enum<T> & Operation>` `Collection<? extends Operation>` 제네릭 패턴을 활용하면
>   타입 안정성과 유연성을 모두 확보할 수 있다.
> * **즉, “확장 가능한 열거 타입을 만드는 유일한 방법”은 인터페이스를 사용하는 것이다.**


## 1. 배경: 열거 타입의 본질적 한계

* `enum`은 자바에서 **상속이 불가능한 클래스**다.
  즉, 기존 열거 타입을 **확장(extend)** 할 수 없다.
* 대부분의 경우 문제가 없지만,
  **“열거 상수를 외부에서 추가해야 하는 경우”**엔 제약이 된다.

예를 들어 `Operation`이라는 산술 연산 열거 타입이 있을 때,
나중에 모듈 사용자가 제곱(`^`), 나머지(`%`) 등의 연산을 추가하고 싶다면 불가능하다.

---

## 2. 해결책: 인터페이스를 통해 확장 모방하기

열거 타입을 직접 확장하지 말고,
**공통 인터페이스를 정의하고 각 enum이 이를 구현하도록 설계**한다.

```java
public interface Operation {
    double apply(double x, double y);
}
```

### 기본 연산(enum)

```java
public enum BasicOperation implements Operation {
    PLUS("+")  { public double apply(double x, double y) { return x + y; } },
    MINUS("-") { public double apply(double x, double y) { return x - y; } },
    TIMES("*") { public double apply(double x, double y) { return x * y; } },
    DIVIDE("/") { public double apply(double x, double y) { return x / y; } };

    private final String symbol;
    BasicOperation(String symbol) { this.symbol = symbol; }
    @Override public String toString() { return symbol; }
}
```

### 확장 연산(enum)

```java
public enum ExtendedOperation implements Operation {
    EXP("^")   { public double apply(double x, double y) { return Math.pow(x, y); } },
    REMAINDER("%") { public double apply(double x, double y) { return x % y; } };

    private final String symbol;
    ExtendedOperation(String symbol) { this.symbol = symbol; }
    @Override public String toString() { return symbol; }
}
```

→ 이렇게 하면 `BasicOperation`, `ExtendedOperation` 모두
`Operation` 타입으로 다룰 수 있다.

---

## 3. 타입 안전성을 확보하는 제네릭 한정

인터페이스로 확장 가능하게 만들었더라도,
메서드에서 여러 열거 타입을 **타입 안전하게 다루는 법**이 필요하다.

이를 위해 다음과 같은 제네릭 한정식을 사용한다:

```java
private static <T extends Enum<T> & Operation> void test(
        Class<T> opEnumType, double x, double y) {

    for (Operation op : opEnumType.getEnumConstants()) { // 클래스가 이넘 타입이면 반환, 아니면 null
        System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
    }
}
```

### 문법 해석

* `T extends Enum<T>` → T는 열거 타입이다.
* `& Operation` → 동시에 Operation 인터페이스를 구현해야 한다.
* 즉, “**열거 타입이면서 Operation을 구현한 타입만**” 받을 수 있다.

### 예시

```java
test(BasicOperation.class, 2, 4);
test(ExtendedOperation.class, 2, 4);
```

→ `BasicOperation`, `ExtendedOperation` 모두 허용된다.

---

## 4. 와일드카드를 통한 유연한 활용

만약 여러 열거 타입을 섞어서 한 번에 다루고 싶다면,
`Collection<? extends Operation>` 형태로 받을 수 있다.

```java
private static void test(Collection<? extends Operation> operations,
                         double x, double y) {
    for (Operation op : operations)
        System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
}

test(Arrays.asList(BasicOperation.values()), 2, 4);
test(Arrays.asList(ExtendedOperation.values()), 2, 4);
```

* `<T extends Enum<T> & Operation>` : **클래스 단위 제약 (정적 타입 안전성 확보)**
* `<? extends Operation>` : **인스턴스 단위 유연성 확보**

| 구분                                | 목적                  | 예시                                       |
| --------------------------------- | ------------------- | ---------------------------------------- |
| `<T extends Enum<T> & Operation>` | 열거 타입 클래스 단위로 순회할 때 | `test(BasicOperation.class, …)`          |
| `<? extends Operation>`           | 여러 열거 타입을 묶어서 다룰 때  | `test(List.of(BasicOperation.values()))` |

---

## 5. 제한 사항

* 인터페이스 기반 확장은 **EnumSet**, **EnumMap** 같은 열거 전용 유틸리티를 사용할 수 없다.
  (`EnumSet<Operation>`은 불가능 → `Set<Operation>` 사용 필요)
* `switch` 문에서도 사용할 수 없다.
  (열거 상수의 집합이 고정되어 있지 않기 때문)
* 따라서 “확장 가능성”을 얻는 대신 “컴파일타임 완전성”은 잃게 된다.
