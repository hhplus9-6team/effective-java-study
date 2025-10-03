# ITEM 02. 생성자에 매개변수가 많다면 빌더를 고려하라

## 1. 서론: 문제 제기

```java
// 생성자를 이용한 객체 생성
NutritionFacts cocaCola = new NutritionFacts(240, 8, 100, 0, 35, 27);

// 정적 팩토리 메서드를 이용한 객체 생성
NutritionFacts cocaCola = NutritionFacts.of(240, 8, 100, 0, 35, 27);
```

자바에서 객체 생성을 위한 방법은  **1) 정적 팩터리 메서드**와 **2) 생성자** 로 두 가지가 존재합니다.

하지만 매개변수 수가 늘어날수록 두 방식 모두 가독성이 떨어지고 잘못된 순서로 값을 넘겨도 컴파일러가 잡아주지 못해 안정성이 낮아집니다.

<br>

## 2. 기존 접근법과 한계

매개변수 수가 많은 객체 생성하기 위해 과거에는 두 가지 방법을 주로 사용했습니다.

### 1) 점층적 생성자 패턴 (Telescoping Constructor)

```java
public class NutritionFacts {
    
    ...
    
    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }
    
    public NutritionFacts(int servingSize, int servings, int calories) {
        this(servingSize, servings, calories, 0);
    }
    
    public NutritionFacts(int servingSize, int servings, int calories, int fat) {
        this.servingSize = servingSize;
        this.servings = servings;
        this.calories = calories;
        this.fat = fat;
    }
}

new NutritionFacts(240, 8);
new NutritionFacts(240, 8, 100);
new NutritionFacts(240, 8, 100, 35);
```

첫 번째 방법은 **점층적 생성자 패턴(Telescoping constructor pattern)** 으로, 매개변수의 개수에 따라 생성자를 여러 개 오버로딩하여 정의해두는 방식입니다.

이 방식을 사용하면 객체가 선택적 매개변수를 유연하게 처리할 수 있고, 객체 생성 시점에 일관성을 완전하게 유지할 수 있습니다.

- 일관성 : 객체가 가져야 하는 불변식(invariant), 즉 **“객체가 항상 만족해야 하는 조건/규칙”**을 의미

하지만 매개변수가 많아지면 생성자 수가 기하급수적으로 늘면서 코드를 작성하거나 읽기 어렵고, 인자의 의미가 명확하지 않아 매개변수 순서 착각하기 쉽습니다.

### 2) 자바빈즈 패턴 (JavaBeans pattern)

```java
NutritionFacts cocaCola = new NutritionFacts();
cocaCola.setServingSize(240);
cocaCola.setServings(8);
cocaCola.setCalories(100);
```

두 번째 방법은 **자바빈즈 패턴(JavaBeans pattern)** 으로, 기본 생성자와 setter 메서드로 매개변수를 전달하는 방식입니다. 

이 패턴은 1996년 JavaBeans 컴포넌트 모델에서 비롯되었는데, GUI 컴포넌트를 IDE에서 손쉽게 조작하기 위해 기본 생성자와 getter/setter 메서드 규약을 정의한 것이 시작이었습니다.

이 방식을 사용하면 가독성이 높아지고 선택적 매개변수 조합이 유연합니다.

그러나 이 방식을 사용했을 때는 다음과 같은 단점이 존재합니다.

- 객체가 완전히 초기화되기 전에도 외부에서 접근할 수 있어 일관성(invariant)이 깨집니다.

- setter 메서드 때문에 불변 객체(immutable object)를 만들 수 없습니다.

- 따라서 스레드 안전성도 보장되지 않습니다.

<br>

이러한 단점을 보완하기 위해서는 추가적인 작업이 필요합니다.

```
synchronized, ReentrantLock, volatile
```

스레드 안정성을 보장하기 위해 프로그래머가 동기화(synchronization) 같은 추가 작업을 해줘야 합니다.

```java
public class Config {
    private String url;
    private boolean frozen = false;

    public void setUrl(String url) {
        if (frozen) {
            throw new IllegalStateException("Object is frozen and cannot be modified");
        }
        this.url = url;
    }

    public String getUrl() {
        return url;
    }

    // freeze 메서드: 더 이상 setter 못 쓰게 막음
    public void freeze() {
        this.frozen = true;
    }

    public static void main(String[] args) {
        Config config = new Config();
        config.setUrl("http://localhost");
        config.freeze();

        config.setUrl("http://prod"); // ❌ 예외 발생: 이미 freeze 됨
    }
}
```

혹은 객체를 만들 때는 setter 를 사용하다가 `freeze()` 메서드를 호출하면 더 이상 수정 불가한 상태로 전환하는 프리징 방식을 사용해야 합니다.

그러나 `freeze()` 메서드를 호출했는지 여부를 확인해야 하고 누락/순서 실수 시 런타임 오류가 발생하여 실전에서 쓰기 어렵습니다.

<br>

## 3. 빌더 패턴의 등장

점층적 생성자 패턴과 자바빈즈 패턴으로는 해결되지 않는 문제를 해결하기 위해 **빌더 패턴(Builder pattern)** 을 사용할 수 있습니다.

빌더 패턴은 1994년 GoF의 디자인 패턴에서 ‘복잡한 객체의 생성 절차와 표현을 분리’하는 개념으로 소개되었습니다. 

이후 Joshua Bloch가 Effective Java에서 이 아이디어를 자바의 ‘선택적 매개변수 문제’를 해결하는 방식으로 재해석하여, 우리가 오늘날 흔히 사용하는 형태의 빌더 패턴을 제안했습니다.

### 1) GoF의 Builder 패턴

```java
// Product: 원본 데이터
class DocData {
    String title;
    List<String> sections = new ArrayList<>();
    DocData(String title) { this.title = title; }
    DocData addSection(String text) { sections.add(text); return this; }
}

// Builder 인터페이스: 절차 정의
interface DocBuilder {
    void reset();
    void buildTitle(String title);
    void buildSection(String text);
    String getResult();
}

// ConcreteBuilder 1: HTML 문서
class HtmlDocBuilder implements DocBuilder {
    private StringBuilder sb;
    public void reset() { sb = new StringBuilder(); }
    public void buildTitle(String title) {
        sb.append("<h1>").append(title).append("</h1>\n");
    }
    public void buildSection(String text) {
        sb.append("<p>").append(text).append("</p>\n");
    }
    public String getResult() { return sb.toString(); }
}

// ConcreteBuilder 2: Markdown 문서
class MarkdownDocBuilder implements DocBuilder {
    private StringBuilder sb;
    public void reset() { sb = new StringBuilder(); }
    public void buildTitle(String title) { sb.append("# ").append(title).append("\n\n"); }
    public void buildSection(String text) { sb.append(text).append("\n\n"); }
    public String getResult() { return sb.toString(); }
}

// Director: 동일 절차로 빌더 호출
class DocDirector {
    private DocBuilder builder;
    DocDirector(DocBuilder builder) { this.builder = builder; }
    String construct(DocData data) {
        builder.reset();
        builder.buildTitle(data.title);
        for (String s : data.sections) builder.buildSection(s);
        return builder.getResult();
    }
}
```

GoF에서 제안한 빌더 패턴은 “복잡한 객체의 생성 절차(steps)”와 “그 결과물의 표현(product)”을 분리하는 것이 핵심입니다.

즉, 같은 절차를 따르더라도 다른 표현물을 만들어낼 수 있습니다. 

예를 들어, 동일한 문서 데이터를 가지고 HTML 문서, Markdown 문서, 텍스트 문서를 각각 생성할 수 있습니다.

### 2) Effective Java의 Builder 패턴

```java
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int sodium;

    // private 생성자
    private NutritionFacts(Builder builder) {
        this.servingSize = builder.servingSize;
        this.servings = builder.servings;
        this.calories = builder.calories;
        this.sodium = builder.sodium;
    }

    // static nested Builder
    public static class Builder {
        // 필수 매개변수
        private final int servingSize;
        private final int servings;

        // 선택 매개변수 - 기본값으로 초기화
        private int calories = 0;
        private int sodium = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }

        public Builder calories(int val) { calories = val; return this; }
        public Builder sodium(int val)   { sodium = val; return this; }

        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }
}

NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
        .calories(100)
        .sodium(35)
        .build();
```

반면 Joshua Bloch는 Effective Java에서 전혀 다른 문제를 해결하기 위해 빌더 패턴을 꺼냈습니다.

빌더 패턴은 별도의 `Builder` 클래스를 두고 필수 매개변수는 빌더의 생성자를 통해 받고, 선택적 매개변수는 체이닝 메서드로 설정한 후 `build()` 를 호출해 불변 객체를 완성시키는 것입니다.

이 방식을 사용하면 다음과 같은 장점이 있습니다.

- 각 인자의 의미가 코드에 드러나 가독성이 뛰어납니다.
- 최종 객체는 `build()` 시점에만 생성되므로 일관성이 보장됩니다.
- 불변 객체를 손쉽게 만들 수 있습니다.
- 나아가 필드가 추가되어도 빌더에 메서드를 하나 더 정의하기만 하면 되니 확장성이 좋습니다.

그러나 별도의 `Builder` 클래스 작성 필요하여, 성능적으로 민감한 상황에서는 문제가 발생할 수 있습니다.

<br>

## 4. 빌더 패턴의 확장

### 1) 상속 구조와 빌더

빌더 패턴을 상속 구조에 적용할 때는 부모 클래스의 빌더 메서드가 부모 타입을 반환하기 때문에 체이닝이 끊겨버리는 문제가 있습니다.

이를 해결하기 위해 **재귀적 타입 한정** 과 **셀프 타입 관용구** 기법을 사용합니다.

```java
abstract class PersonBuilder<T extends PersonBuilder<T>> {
    String name;
    T name(String n) { this.name = n; return self(); }
    protected abstract T self();
}

class EmployeeBuilder extends PersonBuilder<EmployeeBuilder> {
    String role;
    EmployeeBuilder role(String r) { this.role = r; return this; }
    @Override protected EmployeeBuilder self() { return this; }
}
```

- 재귀적 타입 한정 : 제네릭 타입 매개변수의 상한(bound)으로 자기 자신을 포함하는 타입을 지정하는 기법

- 셀프 타입 관용구 : 재귀적 타입 한정을 이용해 자기 자신 타입을 반환하도록 강제하는 것


### 2) 공변 반환 타입

```java
class Parent {
    public static class Builder<T extends Builder<T>> {
        abstract Parent build();
    }
}
class Child extends Parent {
    public static class Builder extends Parent.Builder<Builder> {
        @Override
        public Child build() {
            return new Child(this);
        } // 공변 반환
    }
}
```

- 공변 반환 타입 : 메서드 오버라이딩 시 반환 타입을 부모 타입이 아닌 하위 타입으로 반환 

자바 1.4까지는 메서드 오버라이딩 시 부모와 완전히 동일한 타입을 반환했어야 했는데, 자바 5부터는 메서드 오버라이딩 시 반환 타입을 하위 타입으로 좁혀 반환할 수 있게 됐습니다.

이로 인해 상속 관계에서 빌더 패턴을 구현할 때 형변환에 신경쓰지 않고 빌더 사용할 수 있습니다.

<br>

## 5. 불변성과 방어적 복사

빌더 패턴이 강력한 이유 중 하나는 불변 객체를 손쉽게 만들 수 있다는 점입니다.

불변 객체는 스레드 안정성, 예측 가능성, 유지보수성에서 큰 장점을 가집니다.

빌더에서 가변 객체(ex: Date, List, Array 등)를 받았다면 반드시 방어적 복사를 해줘야 합니다.

그렇지 않으면 외부에서 값을 변경할 수 있어 불변식이 깨질 수 있습니다.

### 1) 방어적 복사 없는 예시

```java
public final class Period {
    private final Date start; // java.util.Date는 가변!
    private final Date end;

    private Period(Builder b) {
        // ❌ 그대로 참조를 저장: 외부에서 Date를 바꾸면 여기 상태도 바뀜
        this.start = b.start;
        this.end = b.end;

        // ❌ 검증 시점에도 가변 참조라 TOCTOU(확인-사용 사이 변경) 가능
        if (!start.before(end)) {
            throw new IllegalArgumentException("start must be before end");
        }
    }

    public static class Builder {
        private Date start;
        private Date end;

        public Builder start(Date d) { this.start = d; return this; }
        public Builder end(Date d)   { this.end = d;   return this; }
        public Period build()        { return new Period(this); }
    }

    public Date getStart() { return start; } // ❌ 그대로 노출
    public Date getEnd()   { return end;   } // ❌ 그대로 노출
}


Date s = new Date();                 // 지금
Date e = new Date(s.getTime() + 10); // 10ms 뒤

Period p = new Period.Builder().start(s).end(e).build();

// 빌더에 넘겼던 원본을 바꿔서 불변식 깨뜨리기
e.setTime(s.getTime() - 10); // end < start 로 역전
// p의 내부 상태도 바뀜 (참조 공유) → 불변식 붕괴
```

### 2) 방어적 복사 적용 예시
```java
public final class Period {
    private final Date start;
    private final Date end;

    private Period(Builder b) {
        Date s = new Date(b.start.getTime());
        Date e = new Date(b.end.getTime());
        if (!s.before(e)) throw new IllegalArgumentException();
        this.start = s;
        this.end = e;
    }

    public static class Builder {
        private Date start;
        private Date end;

        public Builder start(Date d) {
            this.start = new Date(d.getTime()); // 입력 시 복사
            return this;
        }
        public Builder end(Date d) {
            this.end = new Date(d.getTime()); // 입력 시 복사
            return this;
        }
        public Period build() { return new Period(this); }
    }
}
```

<br>

## 6. Lombok `@Builder`

실무에서는 빌더를 직접 구현하기보다 Lombok 의 `@Builder` 어노테이션을 활용해 자동 생성하는 경우가 많습니다.

### 1) Lombok `@Builder` 사용 예시

```java
import lombok.Builder;
import lombok.ToString;

@Builder
@ToString
public class User {
    private String name;
    private int age;
    private String email;
}
```

### 2) Lombok이 생성해주는 코드 (컴파일 후)

```java
public class User {
    private String name;
    private int age;
    private String email;

    User(String name, int age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
    }

    // --- Builder 클래스가 자동 생성됨 ---
    public static class UserBuilder {
        private String name;
        private int age;
        private String email;

        UserBuilder() {}

        public UserBuilder name(String name) {
            this.name = name;
            return this;
        }

        public UserBuilder age(int age) {
            this.age = age;
            return this;
        }

        public UserBuilder email(String email) {
            this.email = email;
            return this;
        }

        public User build() {
            return new User(name, age, email);
        }

        public String toString() {
            return "User.UserBuilder(name=" + this.name +
                    ", age=" + this.age +
                    ", email=" + this.email + ")";
        }
    }

    // Builder를 얻는 정적 메서드도 추가됨
    public static UserBuilder builder() {
        return new UserBuilder();
    }
}
```

### 3) Lombok 동작 방식 간단 흐름

1. 자바 컴파일러(javac)가 소스를 컴파일할 때 → Annotation Processing 단계 실행.

2. Lombok이 이 단계에서 동작하면서, @Builder, @Getter 같은 어노테이션을 해석.

3. AST(Abstract Syntax Tree)에 해당하는 메서드, 생성자, 클래스 등을 삽입.

4. 최종적으로는 내가 작성하지 않은 코드가 추가된 상태로 컴파일 결과가 생성됨.

<br>

Lombok은 반복적이고 기계적인 코드를 줄여주는 자바 어노테이션 기반 라이브러리로 개발 생산성을 높이는 것이 목적입니다.

Lombok 에서 제공하는 `@Builder` 를 사용하면 컴파일 타임에 빌더 패턴 코드를 자동으로 생성해줍니다.

그러나 필수 파라미터 강제, 상속 지원이나 유효성 검증 같은 세밀한 제어는 직접 구현해야 합니다.

<br>

## 7. 결론

생성자나 정적 팩터리는 매개변수가 적을 때는 단순하고 빠릅니다. 

하지만 매개변수가 4개 이상으로 많아지고 선택적 인자가 늘어날수록 빌더 패턴이 가독성과 안전성 측면에서 훨씬 낫습니다.

실무에서는 Lombok `@Builder`로 반복 코드를 줄이되, 필수 인자 강제와 불변식 검증은 반드시 직접 챙기는 습관이 필요합니다.

<br>

---

### 질문

Q. Effective Java에서 제안한 Builder 패턴이 점층적 생성자/자바빈즈 패턴에 비해 가지는 장점은 무엇인가요?

Q. 상속 구조에서 빌더 체이닝이 끊기는 문제를 해결하기 위해 사용하는 기법은 무엇인가요?

Q. 빌더에서 가변 객체(Date, List 등)를 다룰 때 **방어적 복사(defensive copy)**가 필요한 이유는 무엇인가요?

Q. Lombok @Builder를 사용할 때 개발자가 직접 챙겨야 할 부분(제한점)은 무엇인가요?

<br>

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 2
- GoF, Design Patterns (1994)
- [JavaBeans Tutorial (Oracle)](https://docs.oracle.com/javase/tutorial/javabeans/)
- [Lombok GitHub](https://github.com/projectlombok/lombok)
- [빌더(Builder) 패턴 - 완벽 마스터하기](https://inpa.tistory.com/entry/GOF-%F0%9F%92%A0-%EB%B9%8C%EB%8D%94Builder-%ED%8C%A8%ED%84%B4-%EB%81%9D%ED%8C%90%EC%99%95-%EC%A0%95%EB%A6%AC)