# ITEM30. 이왕이면 제네릭 메서드로 만들라

> 형변환이 필요한 메서드라면 제네릭으로 만들어라. 제네릭 메서드는 타입 안전성과 재사용성을 모두 잡을 수 있다.

### 배경
- 제네릭 클래스뿐 아니라 메서드도 제네릭으로 만들 수 있다.
- 즉, 입력값과 반환값의 타입을 메서드 내부에서 직접 제어할 수 있다.
- 이렇게 하면 형변환이 필요없고, 컴파일 시점에 타입 안정성을 확보할 수 있다.

### 기존 메서드의 문제점
```
public static Set union(Set s1, Set s2) {
  Set result = new HashSet(s1); // 경고 발생
  result.addAll(s2);            //경고 발생
  return result;
}
```
- 컴파일은 되지만, unchecked 경고 발생
- 이유 : 위의 `HashSet` 과 `addAll` 은 제네릭 형태의 메서드인데 입력값이 타입 매개변수가 아닌 raw type 형태로 호출하고 있기 때문이다.

### 제네릭 메서드로 개선
- 입력과 반환의 원 타입을 타입 매개변수로 지정한다.
- 메서드 안에서도 이 타입 매개변수만 사용하게 수정한다.
```
public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
  Set<E> result = new HashSet(s1);
  result.addAll(s2);
  return result;
}
```
```
public static void main(String[] args) {
  Set<String> guys = Set.of("톰", "딕", "해리");
  Set<String> stooges = Set.of("래리", "모에", "컬리");
  Set<String> aflCio = union(guys, stooges); //안전하게 동작
```
- 장점
  - 클라이언트 코드에서 형변환 필요없음
  - 컴파일 시점에 타입 검사 가능하여 타입 안정성 보장
- 단점
  - 현재는 세 집합(guys, stooges, all)의 타입이 모두 동일해야 한다.
  - 이후 한정적 와일드카드(Item 31) 를 사용하면 더 유연하게 개선 가능


### 제네릭 싱글턴 팩터리 패턴
> 하나의 객체를 여러 타입으로 재사용할 수 있게 만드는 패턴

#### 배경
- 때때로 불변 객체를 여러 타입으로 활용할 수 있게 만들어야 할 때가 있다.
  - 불변 객체라는 것은 인스턴스화가 되면 더 이상 바뀌지 않기에 로직도 일정하다.
  - 근데 이런 로직을 사용할 때 여러가지 타입에서 사용할 수 있도록 하고 싶을 때가 있다.
- 제네릭은 런타임에 타입 정보가 소거되므로 하나의 객체를 어떤 타입으로든 매개변수화할 수 있다.
- 단, 요청한 타입 매개변수에 맞게 타입을 바꿔주는 정적 팩터리가 필요하며, 이를 `제네릭 싱글턴 팩터리`라고 한다.

#### 예시1 - Comparator.reverseOrder()
```
public static <T> Comparator<T> reverseOrder() {
  //T 타입으로 캐스팅 된 Singleton ReverseCompartor 반환, 타입 정보는 소거됨으로 그때그떄 캐스팅만 된다.
  return (Compartor<T>) ReverseCompartor.REVERSE_ORDER;
}
```
- 제네릭 팩터리 메서드가 Singleton 객체를 타입에 맞게 캐스팅해 반환한다.
- 위와 같은 방식으로 불변 객체를 타입에 맞게 여러번 재사용 할 수 있다.

#### 예시2 - 항등함수
- 항등함수란 입력을 그대로 반환하는 함수 `(f(x) = x)`
- 어떤 타입 T이든 `t -> t` 는 상태가 없으로 타입 안전하다.
- 따라서 하나의 싱글턴 객체로 모든 타입에서 재사용 가능하다.
```
public class GenericSingletonFactory {
    private static UnaryOperator<Object> IDENTITY_FN = (t) -> t;

    @SuppressWarnings("unchecked")
    public static <T> UnaryOperator<T> identityFunction() {
        return (UnaryOperator<T>) IDENTITY_FN;
    }
```
- T가 어떤 타입이든 UnaryOperator<Object>는 UnaryOperator<T>가 아니다 → 경고 발생
- 그러나 항등함수는 입력을 그대로 반환하므로 실제로 타입 안전하다.
- 따라서 @SuppressWarnings("unchecked") 를 사용해 경고를 억제해도 된다
- 실제로 자바에서는 Function.identity() 가 같은 역할을 한다.

### 재귀 타입 한정
> 자기 자신이 들어간 표현식을 사용해 타입 매개변수의 허용 범위를 한정할 수 있다.

```
public static <E extends Comparable<E>> E max(Collection<E> c);
```
- 모든 타입 E는 자기 자신과 비교할 수 있어야 한다.
- 즉, Comparable<E> 를 구현한 타입(String, Integer 등)만 허용된다.
- 주로 정렬, 최댓값, 최솟값 등의 연산에서 사용된다.