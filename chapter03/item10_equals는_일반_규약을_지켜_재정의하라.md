# item 10. equals는 일반 규약을 지켜 재정의하라

## 1. Object.equals 의 역할

```java
public class Object {
    public boolean equals(Object obj) {
        return (this == obj);
    }
}
```

- 자바에서 모든 클래스는 `Object` 클래스를 상속받습니다.

- `Object` 클래스가 제공하는 기본 `equals` 메서드는 **참조 동일성(reference equality)** 비교만 수행합니다. 즉, 해당 객체 자체의 주소값이 같은지를 확인한다고 보면 됩니다.

- 그러나 대부분의 값 객체들은 객체 식별성이 아니라 **논리적 동등성** 을 비교해야 할 필요가 있기 때문에 equals 재정의가 필요합니다.
  
  - 예) String, Integer, BigDecimal, LocalDate 등


## 2. equals 재정의가 필요 없는 경우

### 1) 인스턴스가 본질적으로 고유한 경우

- 논리적으로 같은가 여부 자체가 의미 없고, 식별성(==)으로만 비교하면 충분합니다.

- 예) `Thread`, `Socket`, `Database Connection` 같은 자원 관리 객체

### 2) 논리적 동등성 검사가 필요 없는 경우

- Random 객체는 상태(seed)가 있지만, 굳이 “논리적으로 같은 Random”을 비교할 일이 없습니다.

- 예) `Pattern`, `Random`, `System.Logger`

### 3) 상위 클래스 equals가 이미 올바른 경우

- 구현체가 달라도 같은 동등성 규칙이 적용됩니다.

- 예) `List` 구현체들은 `AbstractList`, `Set` 구현체는 `AbstractSet`, `Map` 구현체는 `AbstractMap`이 구현한 equals를 상속받아 사용

```java
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof List))
            return false;

        ListIterator<E> e1 = listIterator();
        ListIterator<?> e2 = ((List<?>) o).listIterator();
        while (e1.hasNext() && e2.hasNext()) {
            E o1 = e1.next();
            Object o2 = e2.next();
            if (!(o1==null ? o2==null : o1.equals(o2)))
                return false;
        }
        return !(e1.hasNext() || e2.hasNext());
    }
}
```

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    
}
```

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    
}

public abstract class AbstractSequentialList<E> extends AbstractList<E> {
    
}
```

### 4) private / package-private 클래스

- 외부에서 equals 호출이 불가능하므로 재정의가 불필요합니다.

### 5) 인스턴스 통제 클래스

- 인스턴스 통제 클래스는 객체 생성을 통제하므로, 같은 값을 표현하면 같은 인스턴스 반환하여 `==` 와 `equals` 가 동일하게 동작합니다.

- 예) `Boolean`, `Integer`

```java
Boolean b1 = Boolean.valueOf(true);
Boolean b2 = Boolean.valueOf(true);
System.out.println(b1 == b2);      // true
System.out.println(b1.equals(b2)); // true
```

### 6) Enum 타입

- enum 상수는 JVM 내에서 유일한 인스턴스로 보장되므로, `==` 와 `equals` 가 동일하게 동작합니다.

```java
enum Color { RED, BLUE }
System.out.println(Color.RED == Color.RED);      // true
System.out.println(Color.RED.equals(Color.RED)); // true
```

## 3. equals의 일반 규약

### Javadoc 에서 `Object.equals` 명세 원문

> The equals method implements an equivalence relation on non-null object references:
> 
> - It is reflexive: for any non-null reference value x, x.equals(x) should return true.
> - It is symmetric: for any non-null reference values x and y, x.equals(y) should return true if and only if y.equals(x) returns true.
> - It is transitive: for any non-null reference values x, y, and z, if x.equals(y) returns true and y.equals(z) returns true, then x.equals(z) should return true.
> - It is consistent: for any non-null reference values x and y, multiple invocations of x.equals(y) consistently return true or consistently return false.
> - For any non-null reference value x, x.equals(null) should return false.

- 컴파일러 차원에서 강제하지는 않습니다.

  - equals를 저 규약과 다르게 구현해도 컴파일 에러는 나지 않습니다. 즉, 문법적인 강제력은 없습니다.

- 하지만 자바 언어와 표준 라이브러리 차원에서는 사실상 강제입니다.

  - HashSet, HashMap, List.contains 등 수많은 컬렉션이 equals 규약을 지킨다는 전제 하에 동작합니다.

  - 규약을 위반하면 런타임에서 예상치 못한 버그가 터질 수 있습니다. 
    
    - 예: contains가 true여야 하는데 false가 나오거나, HashSet이 중복을 걸러내지 못함


### 1) 반사성 (Reflexive)

- 어떤 객체는 자기 자신과 같아야 합니다.

- `x.equals(x)`는 항상 true

#### 반사성 위반 예시
```java
class BrokenReflexive {
  private final int id;

  BrokenReflexive(int id) { this.id = id; }

  @Override
  public boolean equals(Object o) {
      return false; // ❌ 자기 자신과도 false
  }
}

BrokenReflexive br = new BrokenReflexive(1);
System.out.println(br.equals(br)); // false ❌ 반사성 위반
```
- 일부러 어기는 것이 아니면 어기기 쉽지 않음

### 2) 대칭성 (Symmetric)

- `x.equals(y)`가 true면 `y.equals(x)`도 true

#### 대칭성 위반 예시

```java
class CaseInsensitiveString {
    private final String s;

    CaseInsensitiveString(String s) { this.s = s; }

    @Override
    public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString)
            return s.equalsIgnoreCase(((CaseInsensitiveString) o).s);
        if (o instanceof String) // ❌ 잘못된 설계
            return s.equalsIgnoreCase((String) o);
        return false;
    }
}

public final class String
        implements java.io.Serializable, Comparable<String>, CharSequence,
        Constable, ConstantDesc {

    public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        return (anObject instanceof String aString)
                && (!COMPACT_STRINGS || this.coder == aString.coder)
                && StringLatin1.equals(value, aString.value);
    }
}

CaseInsensitiveString cis = new CaseInsensitiveString("java");
String s = "java";

System.out.println(cis.equals(s)); // true
System.out.println(s.equals(cis)); // false ❌ 대칭성 위반
```

- `cis.equals(s)` → CaseInsensitiveString 에서 String도 허용하는 잘못된 equals 구현 때문에 true

- `s.equals(cis)` → String의 equals는 매개변수가 반드시 String일 때만 비교하므로 false

### 3) 추이성 (Transitive)

- `x.equals(y)`와 `y.equals(z)`가 true면 `x.equals(z)`도 true

#### 추이성 위반 예시

```java
class Point {
    final int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return p.x == x && p.y == y;
    }
}

class ColorPoint extends Point {
    final String color;
    ColorPoint(int x, int y, String color) {
        super(x, y);
        this.color = color;
    }

    @Override
    public boolean equals(Object o) {
        // 1) Point 계열만 상대함
        if (!(o instanceof Point)) return false;

        // 2) 상대가 순수 Point면 "색을 무시"하고 좌표만 비교 (잘못된 시도)
        if (!(o instanceof ColorPoint)) {
            return super.equals(o); // 좌표만 비교 → red/blue 상관없이 true 가능
        }

        // 3) 상대가 ColorPoint면 색까지 비교
        ColorPoint cp = (ColorPoint) o;
        return super.equals(o) && this.color.equals(cp.color);
    }
}

ColorPoint p1 = new ColorPoint(1, 2, "red");
Point p2 = new Point(1, 2);
ColorPoint p3 = new ColorPoint(1, 2, "blue");

System.out.println(p1.equals(p2));   // true
System.out.println(p2.equals(p3));  // true
System.out.println(p1.equals(p3)); // false ❌ 추이성 위반
```

#### 무한 재귀 위험

```java
class ColorPoint extends Point {
    final String color;

    // 대칭성 맞추려는 '잘못된' 시도: 상대가 Point지만 ColorPoint가 아니면, 상대 equals에 위임
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) return false;
        if (!(o instanceof ColorPoint)) {
            return o.equals(this); // 👈 상대에게 떠넘김
        }
        ColorPoint cp = (ColorPoint) o;
        return super.equals(o) && this.color.equals(cp.color);
    }
}
```

- 많이들 추이성/대칭성 충돌을 회피하려고 상대 타입이 다르면 상대의 equals(this)로 위임하는 타협안을 쓰면서 **무한 재귀** 에 빠질 위험이 발생합니다.

```java
class SmellPoint extends Point {
    final String smell;

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) return false;
        if (!(o instanceof SmellPoint)) {
            return o.equals(this); // 👈 ColorPoint와 '서로 떠넘기기'가 일어남
        }
        SmellPoint sp = (SmellPoint) o;
        return super.equals(o) && this.smell.equals(sp.smell);
    }
}

ColorPoint  red   = new ColorPoint(1, 2, "red");
SmellPoint  mint  = new SmellPoint(1, 2, "mint");

// 서로의 equals를 호출하며 무한히 떠넘김
red.equals(mint);
```

- Point의 다른 하위 클래스가 하나 더 있다고 했을 때, `ColorPoint` 와 `SmellPoint` 모두 상대가 본인 타입이 아니면 상대에게 물어보는 전략을 취하고 있습니다.

- **ColorPoint ↔ SmellPoint** 비교가 시작되면 서로 `equals`를 무한히 재귀 호출하여 StackOverflowError로 터집니다.

#### 해결방안

(1) getClass 사용으로 상/하위 동치성 금지

```java
@Override
public boolean equals(Object o) {
    if (o == this) return true;
    if (o == null || o.getClass() != this.getClass()) return false;
    ColorPoint cp = (ColorPoint) o;
    return super.equals(cp) && color.equals(cp.color);
}
```

- `instanceof` 대신 `getClass()`로 정확히 같은 클래스끼리만 equals 허용
- 추이성은 지키지만 다형성 기대와는 상충
  - 리스코프 치환 원칙 : 어떤 타입의 모든 메서드가 하위 타입에서도 똑같이 잘 작동해야 합니다.
  - 다형성 관점(LSP)에서는 Point와 ColorPoint가 비교 가능하길 바라는데, getClass를 쓰면 그 기대와 상충합니다.

(2) 상속 대신 컴포지션 권장

```java
final class ColorPoint {
    private final Point point;
    private final String color;

    ColorPoint(int x, int y, String color) {
        this.point = new Point(x, y);
        this.color = color;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof ColorPoint)) return false;
        ColorPoint cp = (ColorPoint) o;
        return point.equals(cp.point) && color.equals(cp.color);
    }

    // Point 관점의 비교가 필요하면 별도 메서드 제공
    public boolean sameLocation(Point p) { return this.point.equals(p); }
}
```

- ColorPoint가 Point를 “포함”하도록 하고, equals는 자기 타입끼리만 비교(색 포함), Point와의 비교는 위임/별도 API로 분리


#### 실전 사례: Timestamp vs Date

- `java.sql.Timestamp`는 `java.util.Date`를 확장하며 나노 정밀도를 추가했습니다. 
- 이로 인해 `Date.equals`(밀리초 비교)와 `Timestamp.equals`(나노 비교)의 기준이 달라져 `d.equals(ts)`는 true, `ts.equals(d)`는 false가 되는 대칭성 위반이 발생하게 됩니다. 
- 구체 값 타입을 상속해 비교 기준을 넓히거나(정밀도 추가) 바꾸는 설계는 피해야 합니다. 
- 같은 문제를 예방하려면 상속 대신 컴포지션을 사용하고, 동등성 기준은 동일 타입 내에서만 정의해야 합니다.

### 4) 일관성 (Consistent)

- 값이 변하지 않는 한 결과는 항상 동일해야 합니다.

#### 일관성 위반 예시

```java
class RandomEquals {
    @Override
    public boolean equals(Object o) {
        // ❌ 호출할 때마다 랜덤 결과
        return Math.random() > 0.5;
    }
}

RandomEquals a = new RandomEquals();
RandomEquals b = new RandomEquals();

System.out.println(a.equals(b)); // true
System.out.println(a.equals(b)); // false ❌ 일관성 위반
```

- equals 의 판단에 신뢰할 수 없는 자원이 끼어들어서는 안됩니다.
- equals는 항시 메모리에 존재하는 객체만을 사용한 결정적 계산만 수행해야 합니다.
  - 결정적(deterministic): 같은 입력이 주어졌을 때, 언제 실행해도 항상 같은 출력을 내는 계산. 
  - 비결정적(nondeterministic): 같은 입력을 넣어도 실행할 때마다 결과가 달라질 수 있는 계산.

### 5) null-아님 (Non-null)

- 어떤 객체도 null과 같을 수 없습니다.

- `x.equals(null)`은 항상 false.

#### null-아님 위반 예시

```java
class BrokenNullCheck {
    private final String name;
    BrokenNullCheck(String name) { this.name = name; }

    @Override
    public boolean equals(Object o) {
        // ❌ null이 들어와도 true 반환
        return true;
    }
}

BrokenNullCheck b = new BrokenNullCheck("java");
System.out.println(b.equals(null)); // true ❌ 규약 위반
```

- 수많은 클래스가 입력이 null인지 확인하는 명시적 null 검사로 자신을 보호하지만, `instanceof` 사용 시 묵시적 null 검사로 충분합니다.


## 4. equals 구현 시 유의사항

### 기본 체크리스트

Joshua Bloch는 equals 재정의 시 다음 순서를 따르라고 권장합니다.

1. **== 연산자로 자기 자신 참조인지 확인**
    - 성능 최적화: `if (this == o) return true;`

2. **instanceof 연산자로 타입 확인**
    - 올바른 타입이 아니면 false.
    - `getClass()`와 `instanceof` 중 선택은 신중히:
        - `getClass()` → 상속 간 동등성 금지, 추이성 보장 (하지만 다형성 기대와 충돌 가능).
        - `instanceof` → 다형성 유지, 하지만 하위 클래스가 필드를 추가하면 추이성 깨질 수 있음.

3. **형변환 (cast)**
    - 입력을 올바른 타입으로 안전하게 변환.

4. **핵심 필드 비교**
    - equals에서 "논리적 동등성"을 정의하는 데 중요한 필드만 비교.
    - 성능을 위해 **차이가 잘 나는 필드부터 비교**하는 것이 좋음.
    - 예: 대부분 다를 가능성이 큰 ID, key 필드를 먼저 검사.

### float, double 비교 시 주의

- `==` 연산은 부동소수점 특수값(`NaN`, `+0.0`, `-0.0`) 때문에 문제를 일으킬 수 있음.

- 따라서 **`Float.compare(f1, f2)`**, **`Double.compare(d1, d2)`** 사용을 권장.

#### 오토박싱 주의

(1) Float.equals

```java
public final class Float extends Number implements Comparable<Float> {
private final float value;

    @Override
    public boolean equals(Object obj) {
        return (obj instanceof Float)
            && (floatToIntBits(((Float)obj).value) == floatToIntBits(value));
    }
}

```

- equals의 시그니처는 equals(Object) 입니다.

- float 원시값끼리 비교하는 게 아니라, Object → Float로 캐스팅 후 내부 value를 꺼내서 비교합니다.

- 즉, Float.equals를 쓰려면 float 값을 Float 객체로 박싱해야 호출 가능 → 오토박싱 발생.

(2) Double.equals

```java
public final class Double extends Number implements Comparable<Double> {
private final double value;

    @Override
    public boolean equals(Object obj) {
        return (obj instanceof Double)
            && (doubleToLongBits(((Double)obj).value) == doubleToLongBits(value));
    }
}
```

- 구조는 Float.equals와 똑같습니다.

- double 원시값을 비교하려면 역시 Double로 박싱해야 하므로 오토박싱이 일어납니다.

(3) Objects.equals 

```java
public final class Objects {
    public static boolean equals(Object a, Object b) {
        return (a == b) || (a != null && a.equals(b));
    }
}

```

- Objects.equals는 두 Object를 받아서 null-safe하게 비교해주는 유틸리티입니다.

- 하지만 시그니처가 Object 타입이라, 원시값(int, float, double…)을 넣으면 자동으로 박싱되어 Integer, Float, Double 같은 객체로 바뀐 뒤 호출됩니다.

- 즉, 원시 타입 비교에 사용하면 성능상 불필요한 오토박싱이 발생합니다.


### 비교 순서가 성능에 미치는 영향

- equals는 종종 컬렉션의 키 비교 등에서 수천~수만 번 호출됨.

- 따라서 다를 확률이 높은 필드를 먼저 비교하면 빠르게 false 반환 → 전체 성능 향상.

- 예) User 클래스에서 userId 같은 유니크 키를 먼저 비교 → 대부분의 비교가 여기서 끝남.

### 자동 생성 도구 활용
- equals와 hashCode는 반복적으로 작성해야 하고, 실수로 규약을 어기기 쉽습니다.

- 따라서 IDE 자동 생성, Lombok, AutoValue 같은 도구를 활용하는 것도 방법이 될 수 있습니다.

#### 1) AutoValue (Google)

- `@AutoValue` 애노테이션을 붙이면 equals, hashCode, toString을 자동 생성.

- 불변 객체를 쉽게 만들 수 있고, 수동 구현보다 안전. 단, 상속 구조 지원은 제한적.

```java
@AutoValue
abstract class Person {
    abstract String name();
    abstract int age();

    static Person create(String name, int age) {
        return new AutoValue_Person(name, age);
    }
}
```

#### 2) Lombok

- `@EqualsAndHashCode` 애노테이션으로 equals/hashCode 자동 생성.

- 선택적으로 특정 필드만 비교하거나, 상위 클래스 필드 포함 여부 제어 가능.


## 5. 결론

- equals는 반드시 동치 관계 규약(반사성, 대칭성, 추이성, 일관성, null-아님)을 지켜야 합니다.

- equals가 필요 없는 경우: 고유 객체, Random/Pattern 등, AbstractList 등 상위 클래스가 이미 구현, private/package-private 클래스, 인스턴스 통제 클래스, enum.

- 구현 시: == → instanceof → 형변환 → 핵심 필드 비교 순서.

- 부동소수점은 compare 사용, 원시 타입은 오토박싱 피하기.

- 성능을 위해 잘 다른 필드부터 비교.

- 실무에서는 Lombok이나 AutoValue 같은 도구를 활용해 실수를 줄이고 생산성을 높이되, 논리적 동등성 정의는 개발자가 명확히 책임져야 합니다.


---

### 질문

Q. equals 재정의가 필요 없는 경우는 언제일까요? (예시 객체 최소 2개)

Q. equals 규약 다섯 가지에 대해 설명해주세요.

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 10
- [Java SE Object.equals 공식 문서](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#equals-java.lang.Object-)