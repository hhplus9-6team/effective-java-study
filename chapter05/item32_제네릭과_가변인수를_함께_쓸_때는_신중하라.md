# item 32: 제네릭과 가변인수를 함께 쓸 때는 신중하라

> [!IMPORTANT]
> - 가변인수(varargs) 메서드는 내부적으로 배열을 사용하고, 제네릭 배열은 런타임에 타입이 소거되어 안전하지 않다.
> - 따라서, 제네릭과 가변인수를 함께 쓸 때는 매우 조심하라.
> - 내부에서 배열을 노출하지 않는다면 `@SafeVarargs`로 안전성을 명시하라.


## 문제 상황 예시
```java
static void dangerous(List<String>... stringLists) {
    List<Integer> intList = List.of(42);
    Object[] objects = stringLists;
    objects[0] = intList;             // 힙 오염 발생!
    String s = stringLists[0].get(0); // 런타임시 ClassCastException 발생 -> 보이지 않는 형변환이 숨어있어서
}
```
- 위 코드의 문제는, 제네릭 배열이 `Object[]`로 변환되어 서로 다른 제네릭 타입을 혼합할 수 있게 되는 것이다.  
- 이로 인해 타입 안정성이 완전히 무너진다.


- 가변인수(varargs)는 호출자가 여러 개의 인수를 넘길 수 있도록 해주는 문법적 편의 기능이다.  
- 하지만 **가변인수는** <u>내부적으로 배열을 사용하기 때문에</u> **제네릭 타입과 함께 사용하면 타입 안정성이 깨질 수 있다.**


### 경고 발생 이유
컴파일러는 이런 메서드 선언에 대해 다음과 같은 경고를 발생시킨다:
```
warning: [unchecked] Possible heap pollution from parameterized vararg type List<String>
```
이 경고는 “힙 오염(heap pollution)”이 발생할 수 있음을 뜻한다.

> 힙 오염(heap pollution)이란?
> - **정의**: 매개변수화된 타입이 런타임에 다른 타입의 객체를 참조하는 상황.
> - 즉, 컴파일러는 `List<String>`이라 믿지만 실제 런타임에는 `List<Integer>`가 들어있는 상태를 말한다.
> - 이런 상황은 곧 `ClassCastException`을 야기한다.

## 의문점
Q. 제네릭 varargs 매개변수를 받는 메서드를 선언할 수 있게 한 이유는? 경고로 끝내는 이유는?
A. 실무에서 매우 유용하기 때문..! (생각보다 심플한 이유)

표준 자바 라이브러리에서도 이런 제네릭+가변인수 메서드가 많이 쓰인다:
- `Arrays.asList(T... a)`
- `Collections.addAll(Collection<? super T> c, T... elements)`
- `EnumSet.of(E first, E... rest)`


## 안전하게 사용하는 방법

제네릭과 가변인수를 함께 사용하려면 **정해진 제약 조건**을 지켜야 한다:

1. **varargs 배열을 수정하지 않는다.**
   - 배열에 새로운 요소를 넣거나, 인자를 교체하지 않는다.
2. **varargs 배열을 외부로 노출하지 않는다.**
   - 즉, 배열 자체를 반환하거나, 클래스 필드에 저장하거나, 다른 곳으로 전달하지 않는다.
3. **@SafeVarargs 애너테이션을 반드시 명시한다.**
   - 이 애너테이션은 "이 메서드는 안전하다"고 호출자에게 알려주는 역할을 한다.
   - 다음 조건에서만 사용 가능:
     - Java 8: `static`, `final` 메서드
     - Java 9+: `private` 메서드 포함 가능

## 안전한 예시

```java
@SafeVarargs
static <T> List<T> flatten(List<? extends T>... lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```

이 코드는 varargs 배열을 수정하지 않고 외부에 노출하지 않으므로 안전하다    
따라서, `@SafeVarargs` 애너테이션으로 경고를 억제할 수 있다.



## 대안: 컬렉션 사용

가변인수 대신 **List**를 매개변수로 받으면 더 안전하다. (아이템 28)

```java
static <T> List<T> flatten(List<List<? extends T>> lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```

- 이 방법은 배열을 사용하지 않기 때문에 타입 오염이 발생하지 않는다.

## 퀴즈
Q. 다음 중 안전한 제네릭 가변인수 메서드의 조건이 아닌 것은? (이유도 설명)
A) varargs 배열을 수정하지 않는다
B) varargs 배열을 외부로 노출하지 않는다
C) varargs 배열을 clone() 후 반환한다
D) @SafeVarargs로 안전성을 명시한다

