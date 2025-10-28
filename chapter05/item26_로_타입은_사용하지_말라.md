# item26. 로 타입은 사용하지 말라

## 1. 제네릭의 기본 개념

### 1) 타입 매개변수

- 제네릭 클래스나 제네릭 메서드에서 구체적인 타입을 나중에 지정할 수 있도록 만드는 타입 자리 표시자
- `Box<T>` 에서 `T`가 타입 매개변수를 의미
- 타입 매개변수의 종류와 이름 관례

    이름 | 의미 | 예시
    -|-|-
    T | Type (일반 타입) | List\<T>
    E | Element (컬렉션 원소 타입) | List\<E>
    K, V | Key, Value (맵의 키와 값) | Map\<K, V>
    N | Number (숫자 타입) | Comparable\<N>
    R | Return type (리턴 타입용) | Function\<T, R>

### 2) 제네릭 타입

- 타입 매개변수를 사용하는 클래스나 인터페이스
- `Box<T>`는 제네릭 타입, `Box<String>`은 매개변수화된 타입
- 제네릭 관련 용어

    용어 | 영문 용어                   | 예시
    -|-------------------------|-
    매개변수화 타입 | parameterized type      | List\<String>
    실제 타입 매개변수 | actual type parameter   | String
    제네릭 타입 | generic type            | List\<E>
    정규 타입 매개변수 | formal type parameter   | E
    비한정적 와일드카드 타입 | unbounded wildcard type | List\<?>
    로 타입 | raw type | List
    한정적 타입 매개변수 | bounded type parameter | \<E extends Number>
    재귀적 타입 한정 | recursive type bound | \<T extends Comparable\<T>>
    한정적 와일드카드 타입 | bounded wildcard type | List\<? extends Number>
    제네릭 메서드 | generic method | static \<E> List\<E> asList(E[] a)
    타입 토큰 | type token | String.class

#### 제네릭의 하위 타입 규칙

```java
List<Object> objects = new ArrayList<>();
List<String> strings = new ArrayList<>();

objects = strings; // ❌ 컴파일 에러
```

- `List<Object>` 는 `List<String>`의 상위 타입이 아니어서, 두 제네릭 타입 간에는 상속 관계가 성립하지 않음
- 만약 허용된다면 `List<Object>`에 Integer를 추가할 수도 있어서 타입 안정성이 깨짐

## 2. 로 타입 (Raw Type)

### 1) 로 타입이란?

```java
List list = new ArrayList();
```

- 제네릭 타입에서 타입 매개변수를 생략한 형태
- `List<E>` 에서 `List` 가 로 타입을 의미
- Java 5 이전 버전 호환성을 위해 존재
- 타입 정보를 잃기 때문에 컴파일러의 타입 검사를 무력화함

### 2) 로 타입의 위험성

```java
List list = new ArrayList();
list.add("Hello");
list.add(123); // 가능해짐

for (Object o : list) {
    String s = (String) o; // 런타임에 ClassCastException 발생   
}
```

- 컴파일러가 타입 검사를 못해서 컴파일 시점에 오류를 잠을 수 없고, 런타임에 `ClassCastException` 이 터질 수 있음
- 제네릭의 가장 큰 장점인 타입 안정성을 잃어버림

### 3) 로 타입을 써야 하는 상황

#### Class 리터럴

```java
List.class; // ✅
List<String>.class; // ❌ 문법적으로 불가능

```
- `List<String>.class` 는 문법적으로 불가능하므로 `List.class` 만 사용

#### instanceof 검사

```java
if (obj instanceof List<?>) { ... } // ✅
if (obj instanceof List) { ... }    // ✅
```

- 제네릭은 런타임에 타입 소거 되므로 `instanceof`는 타입 파라미터 없이 검사해야 함
  - `instanceof`는 컴파일 타임이 아니라 런타임 중 클래스의 실제 타입 정보를 검사하는 연산
  - JVM은 List가 맞는지는 알 수 있지만, 그 List가 String 리스트인지 Integer 리스트인지는 알 수 없음

## 3. 와일드카드

### 1) 비한정적 와일드카드

```java
static int count(List<?> list) {
    return list.size();
}
```

- 타입을 알 수 없을 때는 로 타입 대신 비한정적 와일드카드 (`?`) 타입 사용
- `List<?>`는 아무 타입의 리스트든 받을 수 있다는 의미로, 읽기 전용이어서 null 외에 원소추가 불가
  - list가 실제로는 `List<String>` 일 수도 있지만, `List<Integer>`일 수도 있으니 안전성을 컴파일러가 판단할 수 없어서 원소 추가 못하게 막음
  - null은 모든 참조 타입에 안전하게 들어갈 수 있으므로 예외적으로 허용
- 원소 추가를 하고 싶으면 제네릭 메서드나 한정적 와일드카드 타입 사용
  - 제네릭 메서드 : 타입 파라미터를 명시적으로 선언해서 메서드 안 타입이 일관된다는 걸 컴파일러에게 알려줌
    ```java
    public static <T> void addElement(List<T> list, T element) {
        list.add(element); // ✅ 타입 안전
    }
    
    List<String> names = new ArrayList<>();
    addElement(names, "Alice"); // OK
    addElement(names, 123);     // ❌ 타입 불일치
    ```
  - 한정적 와일드카드 : 특정 상위/하위 타입 범위 안에서 추가할 수 있게 한정
    ```java
    void addNumbers(List<? super Integer> list) {
        list.add(10);  // ✅ Integer나 그 하위 타입 추가 가능
        list.add(3.14); // ❌ Double은 안 됨
    }
    ```

### 2) 한정적 와일드카드

#### 상한 한정 (`List<? extends Type>`)

```java
List<? extends Fruit> list = new ArrayList<Apple>();
...
list.get(0);  // ✅ 꺼내기는 안전
list.add(new Banana());  // ❌ 넣기는 위험
```

- Type의 하위 타입만 허용
- 읽기 전용에 적합 (Producer)
  - list 의 실제 구현이 ArrayList\<Apple> 인 경우, 꺼내게 되면 무조건 Fruit 이라 가능하지만 넣는 것은 잘못하고 Banana 를 넣는 상황이 발생 가능
  - 따라서 혹시라도 잘못 넣는 경우를 방지하기 위해 아예 모든 추가를 금지

#### 하한 한정 (`List<? super Type>`)

```java
List<? super Orange> list = new ArrayList<Fruit>();
list.add(new Orange());      // ✅ OK (정확히 맞음)
list.add(new NavelOrange()); // ✅ OK (Orange의 하위 타입)
list.add(new Fruit());       // ❌ 불가 (Fruit은 Orange의 부모)
list.add(new Object());      // ❌ 불가 (Object도 Orange의 부모)
Object x = list.get(0);       // ✅ OK (Object로만 안전)
Orange y = list.get(0);       // ❌ 컴파일 에러
```

- Type의 상위 타입만 허용
- 쓰기 전용에 적합 (Consumer)
  - “쓰기 전용” = T를 ‘소비(consume)’하는 쪽으로 쓰기 적합하다는 의미(PECS: Consumer Super). 하지만 상위 타입(Fruit, Object)을 넣을 수 있다는 뜻이 아님
  - 정확한 구체 타입을 모르는 리스트에다가 Orange를 안전하게 넣고 싶을 때 쓰라고 만든 형태
  - add(…) 가능: T 또는 T의 하위 타입만 추가 가능 → `add(new Orange())`, `add(new NavelOrange())` ✅ 
  - add(…) 불가: T의 상위 타입이나 관계없는 타입은 추가 불가 → `add(new Fruit())`, `add(new Object())` ❌ 
  - get(…)은 Object로만 안전: 꺼낼 때 컴파일러가 확신할 수 있는 건 Object 뿐 → `Object o = list.get(0);` ✅, `Orange o = list.get(0);` ❌

## 4. 결론

- 로 타입은 타입 안전성(type safety) 을 포기하는 행위입니다.

- 제네릭이 제공하는 타입 정보와 컴파일 타임 검사 기능을 활용해야 합니다.

- 타입을 몰라도 되는 경우에는 로 타입이 아닌 `List<?>`를 사용하세요.

- 원소 추가나 변경이 필요하다면, 제네릭 메서드나 한정적 와일드카드를 사용하세요.

---

### 질문

Q. 로 타입(Raw Type)은 왜 위험할까요?

Q. 다음 코드에서 컴파일 에러가 발생하는 이유는 무엇일까요?

```java
if (obj instanceof List<String>) { }
```

Q. `List<?>` 와 `List<Object>`의 차이점은 무엇일까요?

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 26