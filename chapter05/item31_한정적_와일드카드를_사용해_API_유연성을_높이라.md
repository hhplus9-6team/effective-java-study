# item 31. 한정적 와일드카드를 사용해 API 유연성을 높이라
> 1. 제네릭은 불공변이므로, 와일드카드를 사용해 유연성을 확보해야 한다.
> 2. **PECS 법칙**을 기억하라: Producer-extends, Consumer-super.
> 3. **public API는 와일드카드를**, **private 구현부는 제네릭 타입 변수**를 사용하라.
> 4. **타입 매개변수가 한 번만 등장한다면 와일드카드로 바꿔라.**
> 5. 와일드카드를 남용하지 말고, **입력 매개변수에서만** 사용하는 것을 원칙으로 하라.
---

## 1. 개요

제네릭은 타입 안전성과 재사용성을 높이지만, 타입 매개변수를 **너무 엄격하게 선언**하면 API의 활용성이 급격히 떨어진다.
이때 **한정적 와일드카드(bounded wildcard)** 를 사용하면 **유연성(flexibility)** 과 **안전성(safety)** 을 모두 확보할 수 있다.

예를 들어 `List<String>`은 `List<Object>`의 하위 타입이 아니다.
즉, **제네릭 타입은 불공변**이므로 `List<Object>`를 요구하는 API에 `List<String>`을 전달할 수 없다.
→ 이런 문제를 해결하기 위해 **와일드카드(`?`)** 를 사용한다.

---

## 2. 불공변성의 문제

다음 코드는 직관적으로 맞지만 컴파일 오류가 발생한다.

```java
List<Number> numbers = new ArrayList<Integer>(); // 컴파일 오류
```

이는 제네릭이 **공변(covariant)** 하지 않기 때문이다.
`Integer`가 `Number`의 하위 타입이라도, `List<Integer>`는 `List<Number>`의 하위 타입이 아니다.

따라서 다음 코드 또한 오류다.

```java
public void process(List<Number> numbers);
List<Integer> list = List.of(1, 2, 3);
process(list); // 불가능
```

이럴 때 와일드카드를 사용하면 해결된다.

```java
public void process(List<? extends Number> numbers);
```

이제 `List<Integer>`, `List<Double>` 등 `Number`의 하위 타입 리스트도 받을 수 있다.

---

## 3. 한정적 와일드카드의 종류

### (1) `? extends T` – 상한 한정 (Upper Bounded Wildcard)

```java
List<? extends Number> numbers;
```

* **Number나 그 하위 타입**의 리스트를 참조 가능.
* **읽기는 허용**, **쓰기(add)는 불가능**.
* 원소 타입이 정확히 무엇인지 알 수 없기 때문.

```java
Number n = numbers.get(0);  // 읽기 가능
numbers.add(1);              // 컴파일 오류
```

### (2) `? super T` – 하한 한정 (Lower Bounded Wildcard)

```java
List<? super Integer> integers;
```

* **Integer나 그 상위 타입**의 리스트를 참조 가능.
* **쓰기(add)는 가능**, **읽기는 Object 타입으로만 가능.**

```java
integers.add(10);           // OK
Object obj = integers.get(0); // 읽으면 Object
```

---

## 4. PECS 원칙 (Producer-Extends, Consumer-Super)

> **“Producer extends, Consumer super”**

즉,

* **생산자(Producer)** → `? extends T`
* **소비자(Consumer)** → `? super T`

와일드카드를 고를 때 **그 컬렉션이 데이터를 생산하는지, 소비하는지**를 판단하라는 뜻이다.

---

### (1) 생산자 예시 (`? extends`)

`Stack` 클래스에 `pushAll` 메서드를 추가한다고 하자.

```java
public void pushAll(Iterable<E> src);
```

이 선언은 `Stack<Number>`에 `Iterable<Integer>`를 전달할 수 없다.
제네릭 타입 불공변성 때문이다.

따라서 다음처럼 수정해야 한다.

```java
public void pushAll(Iterable<? extends E> src) {
    for (E e : src)
        push(e);
}
```

이제 `Stack<Number>`에 `Iterable<Integer>`를 전달할 수 있다.

---

### (2) 소비자 예시 (`? super`)

이번엔 `popAll` 메서드를 보자.

```java
public void popAll(Collection<E> dst);
```

이 경우 `Stack<Integer>`의 원소를 `Collection<Number>`에 담을 수 없다.
역시 타입 불공변성 때문이다.

수정된 버전:

```java
public void popAll(Collection<? super E> dst) {
    while (!isEmpty())
        dst.add(pop());
}
```

이제 `Collection<Number>`, `Collection<Object>` 모두 가능하다.

---

## 5. 와일드카드와 타입 추론의 관계

* **`<T>` 타입 매개변수**는 **메서드 내부에서 타입을 추론하거나 반환할 때** 필요하다.
* **와일드카드(`? extends`, `? super`)**는 **입력 파라미터의 허용 범위를 넓히는** 용도로만 필요하다.

Joshua Bloch의 규칙:

> **“타입 매개변수가 메서드 선언에서 한 번만 등장한다면, 와일드카드로 바꿔라.”**

이유:

* 그 타입은 추론되거나 반환되지 않으므로 이름 붙일 필요가 없다.
* 와일드카드로 바꾸면 호출 가능한 타입 범위가 넓어진다.

---

## 6. public API vs private 헬퍼 메서드

공개 API에는 와일드카드를 사용하고, 내부 구현에서는 구체적인 제네릭을 사용하는 것이 일반적이다.

예를 들어 `swap` 메서드를 생각해보자.

```java
public static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
}

private static <E> void swapHelper(List<E> list, int i, int j) {
    E tmp = list.get(i);
    list.set(i, list.get(j));
    list.set(j, tmp);
}
```

* `public swap()`은 **유연성 확보**를 위해 와일드카드를 사용한다.
* `private swapHelper()`는 **타입 안정성 확보**를 위해 구체 제네릭(`E`)을 사용한다.

---

## 7. 와일드카드의 한계

* 와일드카드는 **복잡한 타입 관계를 표현하기엔 한계**가 있다.
  → 필요 이상으로 사용하면 오히려 가독성이 떨어진다.
* 특히 반환 타입에 등장하면 사용자가 타입 추론을 못 하므로 API 혼란을 초래한다.
* 따라서 **입력 매개변수에 한정적으로 사용하는 것**이 바람직하다.

---
## QnA
- Q. public API에서 제네릭이 아닌 와일드 카드를 사용하는 것이 선호되는 이유는?
- Q. 답있는건 아닌데 다들 PECS 법칙을 어덯게 받아들이셨는지 궁금합니다.
  > `Comparable`은 PECS 법칙에 따르면 `extends`가 되어야할 것 같은데 `super` 사용해야하더라고요.
  > 그래서 저는생산자, 소비자라는 단어가 모호하다고 느껴서 개인적으로 이 법칙 버리고 범위로 구분해야겠다고 결론 내렸습니다.