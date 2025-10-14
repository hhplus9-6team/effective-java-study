# item 23. 태그 달린 클래스보다는 클래스 계층구조를 활용하라
> - “상속을 확장처럼 쓴다고 해서 모두 위험한 것은 아니다.”
> - 상속의 의도가 **구현 재사용**이면 위험하고,
> - **타입 계층 모델링**이면 합리적이다.
> - 즉, 상속을 사용할 때는 **진짜 is-a 관계인지** 항상 점검해야 한다

## 1. 태그 달린 클래스의 문제점

‘태그 달린 클래스(tagged class)’는 클래스 안에 **태그 필드(tag field)** 를 두고,
그 값에 따라 동작을 `switch` 문으로 분기하는 형태를 말한다.

```java
class Figure {
    enum Shape { RECTANGLE, CIRCLE }

    final Shape shape;

    double length;
    double width;
    double radius;

    Figure(double radius) {
        shape = Shape.CIRCLE;
        this.radius = radius;
    }

    Figure(double length, double width) {
        shape = Shape.RECTANGLE;
        this.length = length;
        this.width = width;
    }

    double area() {
        switch (shape) {
            case RECTANGLE: return length * width;
            case CIRCLE: return Math.PI * radius * radius;
            default: throw new AssertionError(shape);
        }
    }
}
```

### 문제점

* **한 클래스에 여러 개념이 뒤섞임** → 응집도가 낮고, 가독성이 나쁨
* **불필요한 필드 존재** → 사용하지 않는 필드(`radius`, `length`, `width`)가 함께 존재
* **switch 분기 증가** → 새로운 타입 추가 시 모든 메서드에 분기 추가 필요
* **타입 안정성 결여** → 컴파일 시점에 타입별 제약을 강제할 수 없음

---

## 2. 클래스 계층구조로의 리팩터링

이 문제는 **상속을 이용한 다형성** 으로 해결할 수 있다.

```java
abstract class Figure {
    abstract double area();
}

class Circle extends Figure {
    final double radius;

    Circle(double radius) {
        this.radius = radius;
    }

    @Override
    double area() {
        return Math.PI * radius * radius;
    }
}

class Rectangle extends Figure {
    final double length;
    final double width;

    Rectangle(double length, double width) {
        this.length = length;
        this.width = width;
    }

    @Override
    double area() {
        return length * width;
    }
}
```

### 장점

* **타입별 책임이 분리**되어 코드 구조가 명확해진다.
* **새로운 타입 추가 시** switch 문 수정 없이 **클래스 추가만으로 확장 가능**하다.
* **타입 안정성**이 보장된다 (`Circle`에 `length` 필드가 존재하지 않음).

---
## 3. Item 18과의 비교: “상속은 피하라” vs “상속을 활용하라”?

표면적으로는 `InstrumentedSet`(Item 18) 과 `Square`(Item 23) 모두 상속을 사용하여 “확장”을 한 것처럼 보인다.
하지만 두 경우의 핵심적인 차이는 **is-a 관계가 성립하느냐**, 그리고 **상위 클래스의 구현에 의존하느냐**에 있다.



### 1) InstrumentedSet – 구현 재사용을 위한 상속 (잘못된 사례)

```java
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }
}
```

* 목적: `HashSet`의 기능을 **그대로 사용하면서** 카운팅 기능 추가
* 문제점: `HashSet`의 **내부 구현(addAll → add 호출)** 에 의존
* 결과: 상위 클래스의 변경이 하위 클래스의 동작을 깨뜨림
* 관계: 실제로는 “has-a(포함)” 관계인데 “is-a”로 잘못 표현

> 대안: 내부에 `HashSet`을 두고, 필요한 기능을 위임하는 **컴포지션** 방식으로 변경

---

### 2) Square – 타입 계층 모델링을 위한 상속 (올바른 사례)

```java
class Rectangle extends Figure {
    @Override double area() { return length * width; }
}

class Square extends Rectangle {
    Square(double side) { super(side, side); }
}
```

* 목적: “정사각형은 직사각형의 한 종류”라는 **개념적 is-a 관계** 표현
* 상위 클래스(`Rectangle`)의 내부 구현 세부사항에 의존하지 않음
* 단순히 **추상화된 계약(‘면적을 계산한다’)** 을 이행
* override는 “동작 변경”이 아니라 **의미 확장(specialization)**

---

### 3) 두 상속의 본질적 차이

| 구분    | Item 18 – InstrumentedSet | Item 23 – Square |
| ----- | ------------------------- | ---------------- |
| 상속 목적 | 구현 재사용 (기능 덧붙이기)          | 타입 계층 표현 (개념 확장) |
| 관계 성격 | 잘못된 is-a (실제는 has-a)      | 올바른 is-a         |
| 의존 대상 | 상위 클래스의 내부 구현             | 상위 클래스의 추상화(계약)  |
| 안전성   | 상위 클래스 변경에 취약             | 추상화가 유지되면 안전     |
| 대안    | Composition               | 그대로 상속 유지 가능     |

---
## 4. 객체 간의 관계
### 1) is-a 관계 (상속 / Inheritance Relationship)

> **정의:** A가 B의 일종이라면 A is-a B

```java
class Rectangle {
    double area() { return length * width; }
}

class Square extends Rectangle {
    Square(double side) { super(side, side); }
}
```

* 의미: “정사각형은 직사각형의 한 종류다.”
* 구현 도구: **상속(extends)**
* 특징

    * 하위 타입(subtype)은 상위 타입(supertype)의 모든 행동을 지원해야 한다. (LSP – 리스코프 치환 원칙)
    * 상위 타입의 계약(interface or abstract method)에만 의존해야 한다.
    * 상위 클래스의 구현 세부사항에 의존하면 “잘못된 is-a” 가 된다.

> 요약: **타입 계층 모델링을 표현할 때 사용.**
> 예) `Student is-a Person`, `Circle is-a Figure`

---

### 2) has-a 관계 (컴포지션 / Aggregation Relationship)

> **정의:** A가 B를 “가지고 있다”면 A has-a B

```java
class Engine {
    void start() { ... }
}

class Car {
    private Engine engine = new Engine();

    void start() {
        engine.start();
    }
}
```

* 의미: “자동차는 엔진을 가지고 있다.”
* 구현 도구: **필드 (Composition 또는 Aggregation)**
* 특징

    * A는 B의 기능을 내부적으로 사용하지만, A ≠ B이다.
    * 두 객체의 수명 연관에 따라 구분된다.

        * Composition : A가 죽으면 B도 함께 죽는다. (소유)
        * Aggregation : A와 B가 별도 수명을 가질 수 있다. (참조)
* Item 18의 `InstrumentedSet` 에서 권장된 방법이 이 관계다.

  ```java
  class InstrumentedSet<E> implements Set<E> {
      private final Set<E> set; // has-a 관계
      private int addCount = 0;
  }
  ```

> 요약: **구현 재사용을 위해 사용.**
> 예) `Car has-a Engine`, `Team has-a List<Player>`

---

### 3) uses-a 관계 (의존 / Dependency Relationship)

> **정의:** A가 B를 일시적으로 “사용한다”면 A uses-a B

```java
class OrderService {
    void placeOrder(PaymentGateway gateway) {
        gateway.pay();
    }
}
```

* 의미: “주문 서비스는 결제 게이트웨이를 사용한다.”
* 구현 도구: **메서드 파라미터, 로컬 변수, 일시적 참조**
* 특징

    * 짧은 수명 동안만 의존. 객체의 구성요소가 아님.
    * 의존 역전 원칙(DIP)을 통해 느슨하게 연결될 수 있다.

> 요약: **단기적인 협력 관계.**
> 예) `OrderService uses-a PaymentGateway`

---

### 4) 네 관계의 비교

| 구분              | 관계                 | 대표 연결 수단                 | 의미            | 수명 의존성          | 예시                                   |
| --------------- | ------------------ | ------------------------ | ------------- | --------------- | ------------------------------------ |
| **is-a**        | 상속 (Inheritance)   | `extends` / `implements` | “A는 B이다.”     | 상위 변화에 의존 (약함)  | `Square is-a Rectangle`              |
| **has-a**       | 컴포지션 (Composition) | 필드 참조                    | “A는 B를 가진다.”  | A 죽으면 B 죽음 (강함) | `Car has-a Engine`                   |
| **aggregation** | 집합 관계              | 필드 참조                    | “A는 B를 참조한다.” | 별도 수명 (약함)      | `Team has-a Player`                  |
| **uses-a**      | 의존 관계              | 파라미터 / 로컬 참조             | “A는 B를 사용한다.” | 일시적             | `OrderService uses-a PaymentGateway` |

