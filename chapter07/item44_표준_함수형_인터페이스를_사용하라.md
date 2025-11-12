# item44. 표준 함수형 인터페이스를 사용하라

## 1. 함수형 인터페이스

```java
@FunctionalInterface
interface Calculator {
    int apply(int a, int b);
}

Calculator add = (x, y) -> x + y;
System.out.println(add.apply(3, 5)); // 출력: 8
```

- 함수형 인터페이스(Functional Interface) 란 하나의 추상 메서드만을 가진 인터페이스를 말합니다.
- 람다는 본질적으로 "익명 객체를 더 간결하게 표현한 문법"이기 때문에 타입이 없습니다.
- 따라서 람다가 작동하려면 타입을 제공해주는 '함수형 인터페이스'가 필요합니다.

#### 자바 8 이전:

```java
button.addActionListener(new ActionListener() {
    public void actionPerformed(ActionEvent e) {
        System.out.println("클릭!");
    }
});
```
- 자바 8 이전에는 익명 클래스를 이용해 함수처럼 전달해야 했습니다.

#### 자바 8 이후:

```java
button.addActionListener(e -> System.out.println("클릭!"));
```

- 자바 8 에서 람다가 도입되면서 더 간결하고 명확한 의도 전달이 가능해졌습니다.



## 2. 표준 함수형 인터페이스

- 표준 함수형 인터페이스를 사용하면 코드의 일관성과 재사용성이 높아진다.

- 자바 표준 라이브러리(`java.util.function`)에는 대부분의 상황을 커버하는 40여 개의 함수형 인터페이스가 이미 정의되어 있습니다.

- 따라서 새로 만들기 전에 표준 패키지에 이미 있는지 확인이 필요합니다.

### 1) 기본 인터페이스

인터페이스 | 	인자    | 	반환    고 | 함수 시그니처	            | 역할	                        | 예시
-|--------|----------|---------------------|----------------------------|-
UnaryOperator<T> | T      | T        | T apply(T t)        | 인수가 1개인 반환값과 인수 타입 같은 함수 | String::toLowerCase     
BinaryOperator<T> | (T, T) | T        | T apply(T t1, T t2) | 인수가 2개인 반환값과 인수 타입 같은 함수 | BigInteger::add
Predicate<T>	| T	     | boolean  | boolean test(T t)	  | 조건 판별	       | Collection::isEmpty
Function<T,R> | 	T	    | R	       | R apply(T t)        | 입력 → 출력 변환          | 	Arrays::asList
Supplier<T> | 	없음	   | T	       | T get()             | 값을 공급	              | Instant::now         
Consumer<T>	| T	     | void	    | void accept(T t)    | 값을 소비	              | System.out::println

- 가장 많이 쓰이는 표준 함수형 인터페이스

### 2) 기본형 특화 버전

인터페이스	| 설명
-|-
IntSupplier, IntPredicate, IntConsumer	| int 타입용 Supplier / Predicate / Consumer
LongFunction<R>, DoubleUnaryOperator | long, double 입력에 특화된 Function / Operator
ObjIntConsumer<T>, ObjLongConsumer<T>, ObjDoubleConsumer<T> |	객체 + 기본형을 함께 받는 Consumer
BooleanSupplier	| boolean getAsBoolean() 형태의 불린 공급자
ToIntBiFunction<T,U>, ToLongBiFunction<T,U>, ToDoubleBiFunction<T,U>	| 두 객체 입력을 받아 기본형 반환
IntUnaryOperator, LongBinaryOperator 등 |	입력과 출력이 모두 기본형인 연산자

- 제네릭 `Function<Integer, Integer>` 는 오토박싱/언박싱 때문에 성능이 떨어집니다.
- 이를 피하기 위해 기본형 특화 버전이 존재합니다.


### 3) 파생 인터페이스

이름	| 시그니처	| 설명
-|-|-
UnaryOperator<T>	| T -> T	| 자기 자신을 변환
BinaryOperator<T>	| (T,T) -> T	| 두 값을 하나로 결합
BiFunction<T,U,R>	| (T,U) -> R	| 두 입력값을 조합해 변환
BiPredicate<T,U>	| (T,U) -> boolean	| 두 입력값을 비교
BiConsumer<T,U>	| (T,U) -> void	| 두 입력값을 소비


## 3. `@FunctionalInterface`

```java
@FunctionalInterface
interface StringMapper {
    String apply(String s);
    // void log(String s); // ❌ 컴파일 에러
}
```

- 함수형 인터페이스임을 명시적으로 알려주는 애너테이션입니다.
- 컴파일러가 강제로 검증을 해주며, 추상 메서드가 2개 이상이거나 오버로딩으로 규칙 깨질 때 에러 발생시킵니다.
- IDE에서도 자동 완성, 문서화 지원이 강화됩니다.

## 4. 전용 함수형 인터페이스

- 대부분의 경우 표준 함수형 인터페이스로 충분하지만, 그것만으로 표현하기 어려운 특수한 상황에만 직접 정의합니다.
  - 자주 쓰이며, 이름 자체가 용도를 명확히 설명해줄 때
  - 반드시 따라야 하는 규약이 있을 때
  - 유용한 디폴트 메서드를 제공할 수 있을 때

### 1) 의미가 더 명확해야 할 때

```java
@FunctionalInterface
interface PasswordEncoder {
    String encode(String rawPassword);
}
```

- `Function<String, String>` 으로도 표현 가능하지만, `PasswordEncoder`가 훨씬 의도가 분명합니다.

### 2) Checked Exception 이 필요한 경우

```java
@FunctionalInterface
interface ThrowingFunction<T, R> {
    R apply(T t) throws IOException;
}
```

- 표준 함수형 인터페이스는 throws 를 지원하지 않기 때문에 직접 정의해야 합니다.

- IO, DB, Reflection 등 체크 예외를 던질 가능성이 있는 연산에 유용합니다.

## 5. 주의사항

### 1) 함수형 인터페이스를 다중 정의하지 말 것

```java
void execute(Function<String, String> f) { ... }
void execute(UnaryOperator<String> f) { ... }  // ⚠️ 시그니처 모호

execute(s -> s.trim()); // ❌ 컴파일 에러: 둘 다 매칭됨

// 메서드 이름을 분리
void executeFunction(Function<String, String> f) { ... }
void executeOperator(UnaryOperator<String> f) { ... }

// 혹은 명시적 타입 지정
execute((Function<String, String>) s -> s.trim());
```

- 서로 다른 함수형 인터페이스를 같은 위치의 인수로 받는 메서드를 오버로딩하면, 람다를 전달할 때 컴파일러가 어떤 메서드를 호출해야 할지 모호해집니다.

- 이를 해결하기 위해서는 메서드 이름을 다르게 하거나 명시적으로 타입을 지정해 모호성을 없애야 합니다.

### 2) 겹치는 표준 인터페이스를 구분해 사용하라

혼동하기 쉬운 인터페이스 |	차이점
-|-
Predicate<T> vs Function<T, Boolean> |	Predicate는 조건 판별용이며, Function은 일반 변환용입니다. 조건식엔 Predicate를 사용해야 의미가 명확합니다.
Consumer<T> vs BiConsumer<T,U> |	입력 개수가 다릅니다. 실수로 BiConsumer를 쓰면 불필요한 인자를 요구하게 됩니다.
Supplier<T> vs Callable<T> |	둘 다 “값 공급자”지만, Supplier는 예외를 던질 수 없고, Callable은 throws Exception을 지원합니다.

- 코드의 의미를 가장 잘 전달하는 인터페이스를 선택하는 것이 중요합니다.

## 6. 결론

- 람다의 타입은 함수형 인터페이스다.

- 직접 만들지 말고 `java.util.function` 패키지의 표준 인터페이스를 우선 사용하라.

- 반드시 `@FunctionalInterface`로 의도를 명시하라.

- 이름, 시그니처, 예외 처리, 기본형 여부 등 실제 용도에 맞게 선택하라.

- 함수형 인터페이스는 오버로딩하지 말고, 의미가 겹치는 표준 인터페이스도 혼용하지 말라.

---

### 질문

Q. `@FunctionalInterface` 애너테이션의 역할은 무엇인가요?

Q. 직접 인터페이스를 정의해야 하는 상황은 언제일까요?

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 44