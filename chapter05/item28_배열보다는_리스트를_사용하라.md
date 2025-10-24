# item 28. 배열보다는 리스트를 사용하라
> - 배열은 공변(covariant)하고, 제네릭은 불공변(invariant)하다.
> - 이 차이로 인해 배열은 타입 안전하지 않으며, **제네릭을 사용할 때는 배열보다 리스트(List)** 를 사용하는 것이 바람직하다.

---

## 1. 배열과 제네릭의 차이

| 구분                  | 배열 (Array)                                        | 제네릭 (Generics)                                        |
| ------------------- | ------------------------------------------------- | ----------------------------------------------------- |
| **공변성(Covariance)** | `Sub`가 `Super`의 하위 타입이면 `Sub[]`도 `Super[]`의 하위 타입 | 불공변(Invariant). `List<Sub>`는 `List<Super>`의 하위 타입이 아님 |
| **실체화(Reified)**    | 런타임에도 타입 정보 유지                                    | 타입 소거(Erasure). 컴파일 이후 타입 정보 제거                       |
| **타입 검사 시점**        | 런타임 검사 (ArrayStoreException 가능)                   | 컴파일 시점 검사 (타입 안전성 높음)                                 |


### 1) 공변, 불공변, 반공변의 개념

| 용어                      | 관계      | 설명                 | 예시                                   |
| ----------------------- | ------- | ------------------ | ------------------------------------ |
| **공변 (Covariant)**      | 하위 → 상위 | 타입 계층이 같은 방향으로 유지됨 | `Cat[]` → `Animal[]`                 |
| **불공변 (Invariant)**     | 관계 없음   | 서로 다른 타입은 완전히 별개   | `List<Cat>` ≠ `List<Animal>`         |
| **반공변 (Contravariant)** | 상위 → 하위 | 반대 방향으로 관계 유지      | `Consumer<Animal>` → `Consumer<Cat>` |

---

## 2. 배열의 공변성 문제

```java
Object[] arr = new Long[1];
arr[0] = "문자열"; // ArrayStoreException (런타임 오류)
```

* 컴파일러는 허용하지만, 런타임 시 타입 불일치로 예외 발생.
* 즉, **배열은 공변성으로 인해 런타임에 타입 안정성이 깨질 수 있다.**

반면 제네릭은 불공변이므로 컴파일 시점에 막는다.

```java
List<Object> list = new ArrayList<Long>(); // 컴파일 에러
```

→ **배열은 공변이라 위험하고, 제네릭은 불공변이라 안전하다.**

---

## 3. 제네릭 배열 생성이 금지된 이유

```java
List<String>[] stringLists = new List<String>[1]; // 컴파일 에러
```

- 배열은 런타임에도 원소 타입을 알고 있으나, 제네릭은 런타임에 타입 정보가 사라진다(소거).
- 이로 인해 타입 안정성이 깨질 수 있으므로 제네릭 배열 생성은 금지되어 있다.

---

## 4. 제네릭과 배열 혼용의 문제

```java
public class Stack<E> {
    private E[] elements = new E[10]; // 불가능
}
```

→ **제네릭 배열 생성 불가**

### 해결책 1. Object 배열 사용 후 캐스팅

```java
elements = (E[]) new Object[10];
```

* 내부에서만 사용하고 외부로 노출하지 않으면 안전.
* 단, 컴파일 경고(`unchecked cast`) 발생.

### 해결책 2. 리스트로 대체

```java
public class Stack<E> {
    private final List<E> elements = new ArrayList<>();
    public void push(E e) { elements.add(e); }
    public E pop() { return elements.remove(elements.size() - 1); }
}
```

* 타입 안전성 확보
* 크기 조절 용이
* 제네릭 경고 없음

---

## 5. 제네릭에서의 공변/반공변 표현: 와일드카드
> 주의 : `T extends` 가 아니라 와일드카드`?`임

자바 제네릭은 불공변이므로, 상하위 타입 관계를 명시하기 위해 **와일드카드**(`? extends`, `? super`)를 사용한다.

---

### 5.1 `? extends T` – 공변 (Covariant)

> **"T의 하위 타입만 받겠다"**

읽기 전용(Read-only)으로 안전하다.

```java
List<? extends Number> numbers = List.of(1, 2, 3);
Number n = numbers.get(0);  // 읽기는 가능
numbers.add(4);             // 컴파일 에러 (쓰기 불가)
```

* 하위 타입(`Integer`, `Double`)을 담은 리스트를 참조 가능
* 하지만 원소를 추가할 수 없다 (타입 불명확)
* **읽기 전용(Producer)** 로만 사용해야 한다.

---

### 5.2 `? super T` – 반공변 (Contravariant)

> **"T의 상위 타입만 받겠다"**

쓰기 전용(Write-only)으로 안전하다.

```java
List<? super Integer> list = new ArrayList<Number>();
list.add(10);       // 추가 가능
Object o = list.get(0); // 반환 시 Object로만 읽을 수 있음
```
* 상위 타입(`Number`, `Object`)을 담은 리스트를 참조 가능
* **쓰기 전용(Consumer)** 으로 활용할 수 있다.

---

### 5.3 PECS 원칙

> **PECS: Producer Extends, Consumer Super**

| 역할             | 키워드           | 설명                 | 예시                       |
| -------------- | ------------- | ------------------ | ------------------------ |
| Producer (생산자) | `? extends T` | T의 하위 타입에서 값을 꺼낼 때 | `List<? extends Number>` |
| Consumer (소비자) | `? super T`   | T의 상위 타입에 값을 넣을 때  | `List<? super Integer>`  |

즉,

* 데이터를 **꺼낼 때**는 `extends`
* 데이터를 **넣을 때**는 `super`

----
## QnA
- Q. 공변이란?
- Q. 배열과 제니릭 리스트의 차이 3가지?
