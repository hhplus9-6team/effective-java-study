# item 29. 이왕이면 제네릭 타입으로 만들라

> [!IMPORTANT]  
> 클라이언트에서 직접 형변환해야 하는 타입보다 제네릭 타입이 더 안전하고 쓰기 편하다
> 가능하면 제네릭 타입(Generic Type)으로 만들어 타입 안정성과 재사용성을 높이자

## 문제 상황

과거 자바에서는 `Object` 타입으로 여러 타입을 처리

```java
public class Stack {
    private Object[] elements;

    public void push(Object e) { ...}

    public Object pop() { ...}
}
```

이 방식은 **형변환(casting)** 을 직접 수행해야 하므로 오류가 **런타임에 발생**

## 해결책: 제네릭 타입 사용

클래스 선언에 타입 매개변수 **(보통 E)** 를 추가

```java
public class Stack<E> {
    private E[] elements;

    public void push(E e) { ...}

    public E pop() { ...}
}
```

이제 컴파일 시점에 타입 검사가 수행되어, 잘못된 타입 사용을 방지 가능

## ⚠️ 구현 시 주의점

### 제네릭 배열 생성 불가 문제

`E`와 같은 실체화 불가 타입은 배열로 생성할 수 없음(아이템 28)

```java
elements = new E[DEFAULT_CAPACITY];
```

#### 해결 방법 1. Object 배열을 사용하고 캐스팅하기

```java

@SuppressWarnings("unchecked")
public Stack() {
    elements = (E[]) new Object[DEFAULT_CAPACITY];
}
```
- 배열 생성 시 **한 번만** 형변환
- `elements`는 `private`이며, **오직 `E` 타입만 저장하므로 논리적으로 안전**
- 따라서 이 비검사 형변환은 안전하므로 `@SuppressWarnings("unchecked")`로 경고를 숨김(아이템 27)

#### 해결 방법 2. E[] 대신 Object[]로 변경하기

```java
public E pop() {
    if (size == 0)
        throw new EmptyStackException();

    @SuppressWarnings("unchecked") E result = (E) elements[--size];
    elements[size] = null; // 다 쓴 참조 해제하여 메모리 누수 방지
    return result;
}
```
- 배열 생성 시점에는 `Object[]`를 사용하고, 원소를 꺼낼 때 `(E)`로 형변환
- 내부 구현은 `Object[]`, 외부 노출은 제네릭 `E`로 유지해 **캡슐화와 타입 안정성**을 확보
- `@SuppressWarnings("unchecked")`로 경고를 숨김

**첫 번째 방식 vs 두 번째 방식**

| 구분 | 방법 1: Object 배열 + 캐스팅 | 방법 2: E[] 대신 Object[] |
|------|-------------------------------|-----------------------------|
| **형변환 시점** | 배열 생성 시 한 번 | 요소 반환 시마다 |
| **가독성** | 높음, 코드 간결 | 보통 |
| **힙 오염 가능성** | 있음 (런타임 타입 ≠ 컴파일 타입) | 낮음 |
| **타입 안정성** | 내부에서만 보장 | 내부 + 외부 캡슐화 모두 가능 |
| **실무 선호도** | 높음 | 특정 상황에서만 사용 |

## 제네릭 Stack을 사용하는 예
명령줄 인수들을 역순으로 바꿔 대문자로 출력하는 프로그램 
```java
public static void main(String[] args) {
    Stack<String> stack = new Stack<>();
    
    for (String arg : args)
        stack.push(arg);
    while (!stack.isEmpty())
        System.out.println(stack.pop().toUpperCase()); // 형변환 불필요
}
```

## 아이템 28과 아이템 29의 관점
- 아이템 28
  - **배열과 제네릭은 타입 체계가 달라 함께 쓰면 타입 안정성이 깨진다**는 점을 강조 
  - 따라서 외부 API 설계 시에는 배열보다 리스트(List) 를 사용하라고 권장 (API 설계자 관점의 원칙)
- 그러나 자바 언어 자체는 리스트를 기본 문법 수준에서 제공하지 않음
  - ArrayList<E>나 HashMap<K,V> 같은 제네릭 컬렉션조차 내부적으로 배열(Object[])을 사용해 구현
- 아이템 29
  - “배열을 완전히 피할 수는 없으니, 제네릭 타입 내부에서 안전하게 감싸서 사용하라”는 해결책을 제시
  - 즉, 배열을 외부에 노출하지 말고, 비검사 형변환(@SuppressWarnings(“unchecked”))을 내부로 캡슐화하여 타입 안정성과 성능을 모두 확보하자
## 제네릭 타입에서의 타입 매개변수 제약에 관하여

### 제약이 없는 타입 매개변수(unbounded type parameter)
- 대다수의 제네릭 타입은 타입 매개변수에 제약을 두지 않음  
    ```java
    Stack<Object>, Stack<String>, Stack<Integer>
    ```
- 기본 타입은 불가 -> 해결책: 박싱 타입 사용
    ```java
    Stack<int> // 컴파일 오류
    ```
    

### 제약이 있는 타입 매개변수(bounded type parameter)
- 특정 클래스나 인터페이스의 하위 타입만 허용하고 싶을 때 사용
    ```java
    <T extends 상위 타입>
    ```

- 예시: java.util.councurrent.DelayQueue
    ```java
    public class DelayQueue<E extends Delayed> implements BlockingQueue<E> {
        ...
    }
    ```
    - `<E extends Delayed>`는 java.util.councurrent.DelayQueue의 하위 타입만 받는다는 뜻
    - 따라서 DelayQueue 자신과 이를 사용하는 클라이언트는 DelayQueue의 원소에서 형변환 없이 Delayed 클래스 메서드를 호출할 수 있음
        - ClassCastException 발생 가능성 제거
    - 이러한 E를 한정적 타입 매개변수라고함
    - 모든 타입은 자기 자신의 하위 타입이므로 DelayQueue<Delayed>로도 사용 가능

## 질문
Q1. Object 타입이 아닌 제네릭을 사용하는 이유는 무엇인가요?  
Q2. 아이템 28에서는 배열보다 리스트를 쓰라 했는데, 왜 Stack 예제에서는 배열을 쓰나요?
