# item17. 변경 가능성을 최소화하라

### 불변 클래스
- 객체의 상태를 변경할 수 없는 클래스 = 인스턴스 내부 값을 수정할 수 없는 클래스
- 가변 클래스보다 설계, 구현, 사용이 훨씬 쉽고 안전하다
- 대표적인 예: String, wrapper class (Integer, Long, Boolean, ...)

## 불변 클래스 5가지 규칙
### 1. 객체의 상태를 변경하는 메서드(변경자)를 제공하지 않는다
### 2. 클래스를 확장할 수 없도록 한다
- 하위 클래스에서 의도치 않게 객체의 상태를 변경할 수 있기 때문
- 상속을 막는 대표적인 방법: 클래스를 `final`로 선언
    ```java
    public final class Complex {
       private final double re;
       private final double im;
    }
    ```

### 3. 모든 필드를 final로 선언한다
### 4. 모든 필드를 private으로 선언한다
- 필드가 참조하는 가변 객체를 클라이언트에서 직접 접근해 수정하는 것을 막기 위해
- `public final` -> 다음 릴리스에서 내부 표현을 바꾸지 못하므로 권장하지 않음 (아이템15, 16)
### 5. 자신 외에는 내부의 가변 컴포넌트가 접근할 수 없도록 한다
- 클라이언트가 제공한 객체 참조를 가리키게 하면 안되며 접근자 메서드가 그 필드를 그대로 반환해서도 안된다
- 생성자, 접근자, readObject 메서드 방어적 복사본을 만들어 사용해야 한다 (아이템 88)
  - **특히 Serializable 구현 시 readObject/readResolve로 불변식 보호 필수**
```java
// ❌ 나쁜 예: 가변 객체 그대로 반환
public final Date getDate() {
    return date;
}

// ✅ 좋은 예: 방어적 복사  
public final Date getDate() {
    return new Date(date.getTime());  
}
```

## 불변 객체 예시
```java
public final class Complex {
    private final double re;
    private final double im;

    public Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    public double realPart() {
        return re;
    }

    public double imaginaryPart() {
        return im;
    }

    public Complex plus(Complex c) {
        return new Complex(re + c.re, im + c.im);
    }

    public Complex minus(Complex c) {
        return new Complex(re - c.re, im - c.im);
    }

    public Complex times(Complex c) {
        return new Complex(re * c.re - im * c.im, re * c.im + im * c.re);
    }

    public Complex dividedBy(Complex c) {
        double tmp = c.re * c.re + c.im * c.im;
        return new Complex((re * c.re + im * c.im) / tmp, (im * c.re - re * c.im) / tmp);
    }

    @Override
    public boolean equals(Object o) { }

    @Override
    public int hashCode() { }

    @Override
    public String toString() { }
}
```
- 인스턴스 자신을 수정하지 않고 새로운 인스턴스를 생성하여 반환
- 메서드 이름을 동사대신 **전치사를 사용**하여 불변임을 강조
    - 예: `plus`, `minus` 등
    - 참고: BigInteger/BigDecimal은 예외적으로 동사형(`add`, `subtract`) 사용
    - 전치사형 네이밍은 "새 객체를 반환한다"는 의미를 더 명확히 전달

### 프로그래밍 패러다임 비교
- **함수형 프로그래밍**  
  - 피연산자에 함수를 적용해 그 결과를 반환하지만, 피연산자 자체는 그대로인 프로그래밍 패턴 
  - 불변 영역 비율이 높다는 장점

- **절차적 or 명령형 프로그래밍**  
  - 피연산자에 함수를 적용하면 그 피연산자가 변경되는 프로그래밍 패턴
  - 상태 변경으로 인한 부작용이 발생할 수 있다는 단점

---

## 불변 객체의 장점

### 1. 단순하다
- 불변 객체는 **생성된 시점의 상태를 파괴될 때까지 그대로 간직**한다
- 생성 시점에 확립된 불변식이 계속 유지되므로 믿고 사용할 수 있다

### 2. 스레드 안전하여 동기화가 필요 없다
- 여러 스레드가 동시에 사용해도 절대 훼손되지 않아 **안심하고 공유**할 수 있다
- 불변 객체에 대해서는 **그 어떤 스레드도 다른 스레드에 영향을 줄 수 없다**

#### 재사용 전략
따라서, 한번 만든 인스턴스를 **최대한 재활용**하는 것을 권장한다:

1. 상수(public static final)로 제공
```java
public static final Complex ZERO = new Complex(0, 0);
public static final Complex ONE = new Complex(1, 0);
public static final Complex I = new Complex(0, 1);
```

2. 정적 팩터리 메서드로 캐싱
```java
// 박싱된 기본 타입 클래스
Integer.valueOf(127);  // 캐시된 객체 반환

// BigInteger
BigInteger.ZERO;
BigInteger.ONE;
```
- 메모리 사용량과 가비지 컬렉션 비용 감소
- 클라이언트를 수정하지 않고 필요에 따라 캐싱, 인스턴스 풀링 기법 적용 가능

#### 방어적 복사 불필요
- 불변 클래스는 `clone` 메서드나 복사 생성자를 제공하지 않는 게 좋다
- 아무리 복사해도 원본과 똑같으니 의미가 없다
- **주의**: `String` 클래스의 복사 생성자는 자바 초창기에 만들어진 것으로, 되도록 사용하지 말자 (아이템 6)

### 3. 불변 객체끼리 내부 데이터를 공유할 수 있다

```java
// BigInteger 예시
BigInteger ten = BigInteger.valueOf(10);
BigInteger minusTen = ten.negate();  // 부호만 바꾼 새 객체

// 내부적으로 크기(magnitude)는 같은 배열을 공유하고
// 부호(sign)만 반대로 가지는 방식으로 구현
// → 배열을 복사하지 않아 효율적
```

- BigInteger는 내부적으로 값의 **부호(sign)** 와 **크기(magnitude)** 를 따로 표현
- `negate()` 메서드는 크기가 같고 부호만 반대인 새 BigInteger를 생성할 때, 크기 배열은 복사하지 않고 원본을 그대로 공유

### 4. 불변 객체는 다른 불변 객체들의 구성요소로 사용하기 좋다
- 구조가 아무리 복잡해도 **불변식을 유지하기 수월**하다
- **맵의 키**와 **집합(Set)의 원소**로 쓰기에 안성맞춤
    - 맵이나 집합에 담긴 후 값이 바뀌면 불변식이 허물어지는데, 불변 객체는 이런 걱정이 없다

```java
Map<String, Integer> map = new HashMap<>();
map.put("key", 1);  // String은 불변이므로 안전

Set<LocalDate> dates = new HashSet<>();
dates.add(LocalDate.now());  // LocalDate는 불변이므로 안전
```

### 5. 실패 원자성을 제공한다 (아이템76)
- **실패 원자성**: 메서드에서 예외가 발생한 후에도 그 객체는 여전히(메서드 호출 전과 똑같은) 유효한 상태여야 한다
- 불변 객체는 상태가 절대 바뀌지 않으므로, 작업 도중 예외가 발생하더라도 객체가 잠깐이라도 **불일치 상태에 빠질 염려가 없다**

```java
// 불변 객체
try {
    Complex result = complex1.dividedBy(Complex.ZERO);
} catch (ArithmeticException e) {
    // complex1은 여전히 유효한 상태 보장
}

// 가변 객체
try {
    list.get(100);  // IndexOutOfBoundsException
} catch (Exception e) {
    // list의 상태가 변경되었을 수도 있음
}
```


## 불변 객체 단점
### 1. 값이 다르면 반드시 독립된 객체로 만들어야 한다
```java
// BigInteger에서 비트 하나만 바꾸려 해도
BigInteger moby = ...;
moby = moby.flipBit(0);  // 새로운 BigInteger 인스턴스 생성
```

**문제점**
- 값의 가짓수가 많으면 이를 모두 만드는 데 **큰 비용**이 든다
- 원하는 객체를 완성하기까지의 **중간 단계에서 생성되는 객체들이 모두 버려진다**면 성능 문제가 될 수 있다

**해결책**
1. 흔히 쓰이는 다단계 연산을 예측하여 기본 기능으로 제공
   - 다단계 연산을 기본으로 제공하면 중간 객체 생성을 최소화
2. 가변 동반 클래스(companion class) 제공
   - `String` ↔ `StringBuilder`, ~~`StringBuffer`~~
   - 클라이언트가 원하는 복잡한 연산을 정확히 예측할 수 있다면 package-private 가변 동반 클래스로 충분
   - 예측이 어렵다면 public 가변 동반 클래스를 제공하는 것이 최선
    ```java
    StringBuilder sb = new StringBuilder();
    sb.append("Hello");
    sb.append(" ");
    sb.append("World");
    String result = sb.toString();  // 최종적으로 불변 String 생성
    ```
   
## 불변 클래스를 만드는 설계 방법
1. 기본 방법: 클래스를 final로 선언(5가지 규칙의 2번)

2. 더 유연한 방법: 모든 생성자를 private or package-private으로 선언하고 public 정적 팩터리를 제공
```java
public class Complex {
    private final double re;
    private final double im;

    // private 생성자
    private Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    // 정적 팩터리
    public static Complex valueOf(double re, double im) {
        return new Complex(re, im);
    }
}
```
- 외부에서 상속 불가(사실상 final 클래스)
- 내부 구현 숨길 수 있음
- 객체 캐싱 기능 추가해 성능을 향상시킬 수 있음

## ⚠ BigInteger, BigDecimal 주의점 
문제: 이 클래스들은 final이 아니라서 누군가 가변 하위 클래스를 만들 수 있음  
해결: 신뢰할 수 없는 인스턴스는 방어적 복사  
```java
public void doSomething(BigInteger value) {
    // 하위 클래스일 수 있으니 방어적 복사
    BigInteger safe = new BigInteger(value.toString());
}
```
**핵심: 불변 클래스를 매개변수로 받을 때, 하위 클래스 인스턴스라면 가변일 수 있다고 가정하고 방어적으로 사용해야한다.**


## 정리
### 클래스는 꼭 필요한 경우가 아니라면 불변이어야 한다
- getter가 있다고 해서 무조건 setter를 만들지 말자
- 단순 값 객체는 항상 불변으로 만들자
  - 성능때문에 어쩔 수 없다면(아이템 67) 가변 동반 클래스를 public으로 제공하자
### 불변으로 만들 수 없는 클래스라도 변경할 수 있는 부분을 최소한으로 줄이자
> **다른 합당한 이유가 없다면 모든 필드는 `private final`이어야 한다**
### 생성자는 불변식 설정이 모두 완료된, 초기화가 완벽히 끝난 상태의 객체를 생성해야한다
> **생성자와 정적 패터리 외에 어떤 초기화 메서드도 public으로 제공해서는 안된다**


## 질문
Q. 다음 코드의 문제점은 무엇이고, 어떻게 개선할 수 있나요?
```java
String result = "";
for (int i = 0; i < 10000; i++) {
    result = result + i;  // 매번 새로운 String 객체 생성
}
```
