item40: @Override 애너테이션을 일관되게 사용하라.

# @Override 애너테이션을 왜 꼭 써야 하는가

이 내용의 핵심은 간단합니다. 메서드를 재정의할 때 @Override를 붙이면, 실수로 엉뚱한 메서드를 만드는 걸 방지할 수 있다는 거죠.

## 왜 문제가 생기는가?

원문에 나온 Bigram 예제를 다시 보면, 작성자는 equals를 재정의하려고 했습니다. 하지만 실제로는 완전히 새로운 메서드를 만들어버렸죠.

```java
// 잘못된 코드
public boolean equals(Bigram b) {  // 매개변수가 Bigram
    return b.first == first && b.second == second;
}
```

Object의 equals는 이렇게 생겼습니다:

```java
public boolean equals(Object obj) {  // 매개변수가 Object
    // ...
}
```

매개변수 타입이 다르니까 재정의가 아니라 오버로딩이 된 겁니다. 그래서 Set에 넣어도 중복 체크가 제대로 안 되고, 260개가 다 들어가버린 거죠.

## 실제로 어떻게 고치는가?

@Override를 붙이면 컴파일러가 바로 잡아줍니다:

```java
@Override
public boolean equals(Bigram b) {  // 컴파일 에러 발생!
    return b.first == first && b.second == second;
}
```

그럼 우리는 "아, 매개변수가 잘못됐구나" 하고 바로 고칠 수 있습니다:

```java
@Override
public boolean equals(Object o) {  // 이제 제대로 재정의됨
    if (!(o instanceof Bigram))
        return false;
    Bigram b = (Bigram) o;
    return b.first == first && b.second == second;
}
```

## 더 쉬운 예시

비슷한 실수를 보여드릴게요:

```java
class Animal {
    public void makeSound() {
        System.out.println("동물 소리");
    }
}

class Dog extends Animal {
    // 실수: 오타로 메서드명이 다름
    public void makesound() {  // 소문자 s
        System.out.println("멍멍");
    }
}
```

이렇게 하면 makeSound를 재정의한 게 아니라 새 메서드를 만든 겁니다. 하지만 @Override를 붙이면:

```java
class Dog extends Animal {
    @Override
    public void makesound() {  // 컴파일 에러!
        System.out.println("멍멍");
    }
}
```

컴파일러가 "상위 클래스에 이런 메서드 없는데요?"라고 알려줍니다.

## 언제 써야 하나?

기본 원칙: **재정의하는 모든 메서드에 붙이세요.**

예외가 딱 하나 있는데, 추상 메서드를 구현할 때는 안 붙여도 됩니다:

```java
abstract class Shape {
    abstract double getArea();
}

class Circle extends Shape {
    // @Override 없어도 됨 (있어도 상관없음)
    double getArea() {
        return Math.PI * radius * radius;
    }
}
```

왜냐하면 추상 메서드를 구현 안 하면 어차피 컴파일이 안 되거든요. 하지만 붙여도 전혀 문제없고, 오히려 명확해서 좋습니다.

## 인터페이스 구현할 때도 쓸 수 있나?

네, 가능합니다:

```java
interface Drawable {
    void draw();
}

class Rectangle implements Drawable {
    @Override
    public void draw() {  // 인터페이스 메서드 구현에도 사용 가능
        System.out.println("사각형 그리기");
    }
}
```

특히 디폴트 메서드가 있는 인터페이스라면 더욱 유용합니다.

## 디폴트 메서드가 있는 인터페이스에서 @Override가 더 유용한 이유

디폴트 메서드는 인터페이스에 이미 구현이 들어있는 메서드입니다. 그래서 구현 클래스에서 굳이 구현하지 않아도 되죠. 문제는 이게 오히려 버그를 만들 수 있다는 겁니다.

### 구체적인 예시

```java
interface Notification {
    // 디폴트 메서드 - 기본 구현이 있음
    default void send(String message) {
        System.out.println("기본 알림: " + message);
    }
}

class EmailNotification implements Notification {
    // 이메일로 보내도록 재정의하려고 함
    public void send(String msg) {  // 실수: 매개변수명을 다르게 씀
        System.out.println("이메일 발송: " + msg);
    }
}
```

이 코드는 컴파일이 됩니다. 하지만 의도와 다르게 동작하죠:

```java
Notification notify = new EmailNotification();
notify.send("긴급 공지");  // 출력: "기본 알림: 긴급 공지"
```

왜 이메일 발송이 안 되고 기본 알림이 나올까요? 매개변수명을 다르게 써서 재정의가 아니라 새 메서드가 된 겁니다. 그래서 인터페이스의 디폴트 메서드가 그대로 호출된 거죠.

일반 추상 메서드였다면 구현을 안 하면 컴파일 에러가 나서 바로 알 수 있습니다. 하지만 디폴트 메서드는 구현이 있어서 그냥 넘어가버립니다.

### @Override로 방지하기

```java
class EmailNotification implements Notification {
    @Override
    public void send(String msg) {  // 컴파일 에러 발생!
        System.out.println("이메일 발송: " + msg);
    }
}
```

@Override를 붙이면 컴파일러가 "인터페이스에 send(String msg)라는 메서드 없는데요?"라고 알려줍니다. 그럼 우리는 바로 고칠 수 있죠:

```java
class EmailNotification implements Notification {
    @Override
    public void send(String message) {  // 매개변수명 일치
        System.out.println("이메일 발송: " + message);
    }
}
```

### 더 복잡한 경우

디폴트 메서드의 시그니처가 복잡하면 실수할 가능성이 더 커집니다:

```java
interface DataProcessor {
    default List<String> process(Collection<String> data, boolean ignoreEmpty) {
        // 기본 구현
        return data.stream()
                   .filter(s -> !ignoreEmpty || !s.isEmpty())
                   .collect(Collectors.toList());
    }
}

class CustomProcessor implements DataProcessor {
    // 실수: Collection을 List로 잘못 씀
    public List<String> process(List<String> data, boolean ignoreEmpty) {
        // 커스텀 구현
        return data.stream()
                   .map(String::toUpperCase)
                   .collect(Collectors.toList());
    }
}
```

이것도 컴파일은 되지만, 디폴트 메서드를 재정의한 게 아닙니다. Collection과 List는 다른 타입이니까요. @Override를 붙이면 이런 실수를 즉시 잡을 수 있습니다.

### default 정리

디폴트 메서드가 있는 인터페이스에서 @Override가 특히 중요한 이유:

1. **디폴트 메서드는 구현이 선택사항**이라 실수로 재정의를 안 해도 컴파일됨
2. **시그니처를 살짝 틀리게 쓰면** 재정의가 아니라 새 메서드가 되버림
3. **런타임에 의도와 다르게 동작**해도 겉으로 드러나지 않을 수 있음
4. **@Override를 붙이면 컴파일 시점에 확실히 재정의**되는지 체크됨

결국 디폴트 메서드는 "있어도 되고 없어도 되는" 특성 때문에, 실수로 잘못 작성해도 눈치채기 어렵습니다. 그래서 @Override로 명시적으로 표시하는 게 더욱 중요한 거죠.

## 정리

@Override는 그냥 습관적으로 붙이세요. IDE가 자동으로 넣어주기도 하고요. 이걸 붙이면:
- 오타로 인한 실수 방지
- 매개변수 타입 실수 방지
- 코드 읽는 사람이 "아, 이건 재정의구나" 하고 바로 알 수 있음

안 붙여도 되는 경우는 구체 클래스에서 추상 메서드 구현할 때 정도인데, 그래도 붙이는 게 더 명확합니다.

----------

## 이해도 테스트 문제


다음 코드를 보고 물음에 답하세요.

```java
class Parent {
    public void display(Object obj) {
        System.out.println("Parent");
    }
}

class Child extends Parent {
    @Override
    public void display(String str) {
        System.out.println("Child");
    }
}

public static void main(String[] args) {
    Parent p = new Child();
    p.display("Hello");
}
```

**문제: 위 코드의 실행 결과는 무엇인가?**

① "Child"가 출력된다

② "Parent"가 출력된다

③ 컴파일 에러가 발생한다

④ 런타임 에러가 발생한다

⑤ 아무것도 출력되지 않는다

