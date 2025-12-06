# item 61. 박싱된 기본 타입보다는 기본 타입을 사용하라
> 가능하면 기본타입을 사용하자. 기본타입이 사용이 어려운 경우만 박싱타입을 사용하자.

## 1. 기본 타입 vs 박싱된 기본 타입

* 기본 타입: `int`, `long`, `double`, `boolean` 등. 값 자체를 저장.
* 박싱된 기본 타입: `Integer`, `Long`, `Double`, `Boolean` 등. 객체이며 `null` 가능.
* 오토박싱/오토언박싱으로 두 타입은 자유롭게 섞어 쓰일 수 있지만 **비용과 위험**이 있다.

## 2. 박싱된 기본 타입이 문제를 일으키는 이유

### 2.1. **NullPointerException 위험**

* 언박싱 시 null이면 즉시 NPE 발생.

```java
Integer count = null;
int result = count + 1; // NPE
```

### 2.2. **의도치 않은 객체 비교 (==)**

* `==`은 박싱된 타입에서 **동일 객체인지** 비교한다.
* 값 비교에는 반드시 `equals`를 사용해야 한다.

```java
Integer a = 1000;
Integer b = 1000;
System.out.println(a == b); // false
System.out.println(a.equals(b)); // true
```

### 2.3. **성능 저하**

* 박싱/언박싱은 모두 **객체 생성 또는 변환 비용**이 들어간다.
* 특히 루프 안에서 불필요한 오토박싱이 성능 저하를 유발할 수 있다.

```java
Long sum = 0L;
for (long i = 0; i < Integer.MAX_VALUE; i++) {
    sum += i; // 매 반복마다 오토박싱 발생 → 느림
}
```

## 3. 박싱된 기본 타입을 사용해야 하는 경우

### 3.1. **컬렉션의 원소로 사용할 때**

* 컬렉션은 객체만 담을 수 있으므로 기본 타입은 사용할 수 없음.
* 예: `List<int>` 불가 → `List<Integer>` 필요.

### 3.2. **null을 명시적으로 표현해야 할 때**

* 값이 "없음"을 표현해야 한다면 박싱 타입이 필요하다.
* ex. `Integer score = null;`

### 3.3. **제네릭 타입 매개변수일 때**

* 타입 매개변수는 객체 타입만 허용한다.

```java
OptionalInt는 기본 타입을 지원하기 위해 별도로 제공된다.
```