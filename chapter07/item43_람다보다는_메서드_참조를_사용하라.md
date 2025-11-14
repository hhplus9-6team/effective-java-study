# item 43. 람다보다는 메서드 참조를 사용하라

> [!IMPORTANT]
> 메서드 참조 > 람다 > 익명 클래스  
> 람다는 유연하지만, 동일한 의미를 **더 간결하고 읽기 쉽게 표현할 수 있을 때는 메서드 참조를 사용하는 것이 좋다**

## 메서드 참조가 유리한 이유
1. **간결성**  
   람다에서 반복적으로 작성해야 하는 매개변수와 반환식이 사라져 코드가 짧아진다
   ```java
   map.merge(key, 1, (count, incr) -> count + incr); // 람다
   map.merge(key, 1, Integer::sum); // 메서드 참조
   ```

2. **가독성 향상**  
   "이 메서드를 그대로 사용한다"는 의도를 명확하게 전달한다

3. **오류 감소**  
   람다식에서 매개변수 이름을 잘못 쓰거나 로직을 잘못 작성할 여지 자체가 줄어든다

## 메서드 참조가 적합하지 않은 경우
람다식이 **더 직관적이고 간단하게 읽힐 때**는 굳이 메서드 참조로 바꾸지 않아도 된다  
람다식의 매개변수의 이름 자체가 프로그래머에게 좋은 가이드가 되기도 한다.  

주로 **메서드가 람다랑 같은 클래스에 있을때** 이런 경향이 있다
```java
service.execute(GoshThisClassNameIsHumongous::action); // 메서드 참조
service.execute(() -> action(); // 람다
```

## 메서드 참조의 5가지 유형

### 1. 정적 메서드 참조
형식: `ClassName::staticMethod`  
예:
```java
Integer::sum
Math::max
```
람다 `(x, y) -> Integer.sum(x, y)` 형태를 대체


### 2. 한정(bound) 인스턴스 메서드 참조
함수 객체가 받는 인수와 참조되는 메서드가 받은 인수가 똑같은 메서드를 참조하는 방식  
즉, 이미 특정 객체가 정해져 있는 경우

형식: `instance::method`    
예시:
```java
myService::process
```
람다 `() -> myService.process()` 또는 `(x) -> myService.process(x)`를 대체

### 3. 비한정(unbound) 인스턴스 메서드 참조
함수 객체를 적용하는 시점에 수신 객체를 알려준다  
즉, 아직 특정 객체는 없지만 실행 시 제공되는 인스턴스의 메서드를 호출하는 형태

형식: `ClassName::instanceMethod`  
예시:
```java
String::toLowerCase
String::compareTo
```
람다 `(s) -> s.toLowerCase()` 또는 `(a, b) -> a.compareTo(b)`를 대체

### 4. 생성자 참조
`new` 호출을 메서드 참조 형태로 표현  

형식: `ClassName::new`  
예시:
```java
ArrayList::new
User::new
```
람다 `() -> new ArrayList()` 또는 `(x) -> new User(x)`를 대체


### 5. 배열 생성자 참조
배열을 생성하는 함수를 참조하는 방법

형식: `Type[]::new`  
예시:
```java
int[]::new
```
람다 `len -> new int[len]`를 대체


## 결론
- 메서드 참조가 **더 짧고 명확**하면 메서드 참조를 사용하라
- 람다가 **더 자연스럽고 읽기 쉬우면** 람다를 유지하라

## 보충
일반적으로 “람다로 할 수 없는 일은 메서드 참조로도 할 수 없다”는 규칙이 성립하지만,  
**제네릭 함수 타입(generic functional type)을 사용할 때는 예외가 된다.**

예를 들어, 아래처럼 서로 다른 제네릭 메서드를 가진 인터페이스들이 있다고 하자

```java
interface G1 {
    <E extends Exception> Object m() throws E;
}
interface G2 {
    <F extends Exception> String m() throws Exception;
}
interface G extends G1, G2 {}
```

위 두 메서드가 하나의 함수형 메서드로 결합되면서 반환 타입은 `Object`와 `String` 중 더 구체적인 `String`으로 수렴하고,  
예외는 `<E extends Exception>`과 `throws Exception`이 합쳐져 `<F extends Exception> throws F` 형태로 결정된다.

따라서 G의 추상 메서드는 아래와 같이 하나의 함수 타입으로 표현된다:

- **반환 타입:** String
- **타입 매개변수:** F extends Exception
- **예외:** throws F

따라서 G를 함수 타입으로 표현하면  
`<F extends Exception> () -> String throws F`

이 경우 G는 **서명은 같지만 반환 타입·예외·타입 매개변수 등이 서로 다른 두 메서드를 동시에 상속**받는다.  
람다식은 “어떤 m()을 구현해야 하는지” 단일하게 결정할 수 없기 때문에 컴파일이 불가능하다.

그러나 메서드 참조는 먼저 **실제 메서드의 시그니처를 기반으로** G1/G2 중 어떤 m()과 일치하는지를 컴파일러가 역으로 결정할 수 있다.

```java
G g = String::toLowerCase;
```
