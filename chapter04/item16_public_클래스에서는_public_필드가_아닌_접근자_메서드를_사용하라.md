# item 16. public 클래스에서는 public 필드가 아닌 접근자 메서드를 사용하라
> 외부에서 접근 가능한 클래스는 **가변 필드를 공개하지 말라**.
> 
> 하지만, 외부에서 접근할 수 없는 클래스(예: private 중첩 클래스나 package-private 클래스)는 필드를 공개해도 괜찮다.
 
## public 클래스에서 필드를 공개했을 경우 단점
공개된(public) 필드는 아래와 같이 단점이 존재한다.
- final하지 않으면 **불변식을 보장할 수 없다**
- API 변경 없이 표현방식을 수정할 수 없다
- 필드 읽을 때 부수 작업을 수행할 수 없다

### 1. public 필드가 불변식 보장 안되는 이유
> **불변식(invariant)이란?**
>
> 객체가 언제나 만족해야 하는 조건

#### 불변식이 깨지는 예시
```java
public class Rectangle {
    public int width;
    public int height;
}

Rectangle r = new Rectangle();
r.width = -5;   // 불변식(width > 0) 깨짐

```

#### private 필드와 함께 setter을 제공한다면
```java
public class Rectangle {
    private int width;
    private int height;

    public void setWidth(int width) {
        if (width <= 0) throw new IllegalArgumentException();
        this.width = width;
    }

    public void setHeight(int height) {
        if (height <= 0) throw new IllegalArgumentException();
        this.height = height;
    }
}
```
이제 외부에서는 필드를 직접 바꿀 수 없고, setWidth()를 통해서만 수정할 수 있으므로
항상 width > 0, height > 0 이 유지된다.

### 2. public은 표현을 맘대로 바꿀 수 없다
반면 private의 경우는 내부 구현이 바뀌어도 외부는 영향을 안 받는다
```java
public class Rectangle {
    private int x1, y1, x2, y2;

    public void setWidth(int width) {
        if (width <= 0) throw new IllegalArgumentException();
        this.x2 = this.x1 + width;
    }
}
```
나중에 Rectangle을 (x1, y1, x2, y2) 좌표로 바꿔도, 외부에서는 여전히 setWidth()와 setHeight()를 쓰니까
클래스 내부의 표현이 어떻게 되든 불변식은 유지된다.
---
## 예외적으로 public 필드여도 괜찮은 경우
### package-private 클래스
package-private 클래스도 외부 모듈(패키지)에는 드러나지 않기 때문에
캡슐화가 깨지지 않는다
```java
class PackagePrivateClass {
    int value;
}
```
### private 중첩 클래스
클래스가 private 중첩 클래스라면, 외부에서 접근할 수 없기 때문에 필드를 직접 공개해도 문제없다
```java
public class Example {
    private static class Point {
        public double x;
        public double y;
    }
}
```
---
## `public static final` 필드에 대한 저자의 견해 탐구
> “public 필드를 피하라”는 조언은 **객체의 내부 표현을 외부에 노출하지 말라는 뜻**이지,
> “어떤 상수도 public으로 공개하지 말라”는 뜻은 아니다.

### 저자가 말한 "static final"이 옳지 않다라는 문맥
1. **“상수가 가리키는 값이 진짜 불변(immutable)인가?”** 를 기준으로 구분하고 있으며
2. 불변한 상수가 **객체의 속성**인 경우에 한정했다.

---
### 1. `static final`은 불변함을 보장하는가?

* `final`은 **참조 자체를 바꿀 수 없다는 의미**이지,
  그 **참조가 가리키는 객체의 내용까지 불변이라는 뜻은 아니다.**

```java
public static final List<String> NAMES = new ArrayList<>();
```
* `NAMES` 참조는 다른 리스트로 교체할 수 없지만
* 내부 요소(`add`, `clear`)는 얼마든지 변경 가능하다.
  → 결국 외부 코드가 클래스 내부 상태를 바꿀 수 있다.

즉, `final`만으로는 캡슐화가 완전하지 않다.

---
### 2. `static final`이 primitive 타입이면 괜찮은가?
```java
public final class Time {
    private static final int HOURS_PER_DAY = 24;
    private static final int MINUTES_PER_HOURS = 60;
    
    public final int hour;
    public final int minute;
    
    public Time(int hour, int minut){
        // 생략
    }
}
```
- 책에 제시된 `public final` 이 옳지 않은 경우의 예시이다.
- final이면서 primitive 타입이여서 불변함은 보장되지만
- 여전히 public으로 선언했을 때의 단점이 해결되지 않아서 옳지 않다라고 얘기한다
    - API를 변경하지 않고는 표현 방식을 바꿀 수 없다
    - 필드를 읽을 때 부수 작업을 수행할 수 없다
- 여기서 이런 단점이 존재하는 경우는 객체의 "상태"를 나타내는 경우를 말했다

---
### 3. "상수"는 public 필드 OK
```java
public class PhysicalConstants {
    private PhysicalConstants() {}  // 인스턴스화 방지

    public static final double AVOGADROS_NUMBER = 6.022_141_99e23;
    public static final double BOLTZMANN_CONSTANT = 1.380_6503e-23;
}
```
기서 공개되는 것은 **“객체의 속성”이 아니라 “불변의 값”** 이다.

- 인스턴스 상태와 무관하고
- 불변이며 (final)
- 외부에서 수정 불가능 (static, final)
- 클래스의 동작에도 영향을 주지 않는 “순수 데이터”

이건 상태 노출이 아니라 상수 정의이므로, 캡슐화 원칙과 충돌하지 않다.

---
## QnA
- public 클래스에서 public 필드를 사용하면 안좋은 이유
- `public static final` 필드를 만들 때 고려해봐야하는 점