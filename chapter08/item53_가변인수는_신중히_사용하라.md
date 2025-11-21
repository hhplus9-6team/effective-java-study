# item53. 가변인수은 신중히 사용하라

> [!IMPORTANT]
> 가변인수는 메서드에 개수 제한 없이 인수를 전달할 수 있게 해주는 문법이지만, 잘못 사용하면 안전성과 성능 문제가 발생할 수 있어 신중하게 사용해야 한다.

## 가변인수(varargs)란?
- 명시한 타입의 인수를 0개 이상 받을 수 있다
- 가변인수 메서드 호출시, 인수의 개수와 길이가 같은 배열을 만들고 이 배열에 저장하여 가변인수 메서드에 전달한다

```java
static int sum(int... args) {
    int sum = 0;
    for (int arg : args) {
        sum += arg;
    }
    return sum;
}
```

## 문제가 되는 경우
### 1. 최소 인수 개수를 강제하지 못한다

예: 최소 1개의 인수를 받아야 하는 메서드
```java
static int min(int... args) {
    if (args.length == 0) {
        throw new IllegalArgumentException("최소값을 구하려면 인수를 1개 이상 필요합니다.");
    }
    int min = args[0];
    for (int i = 1; i < args.length; i++) {
        if (args[i] < min) {
            min = args[i];
        }
    }
    return min;
}
```
- **인수 개수는 런타임에 배열의 길이로 알 수 있기에** 가변인수 메서드에 인수를 0개 전달하는 경우 런타임 예외가 발생한다

### 2. 성능 문제
- 가변인수 메서드 호출시마다 JVM은 매번 새로운 배열이 생성한다
- 자주 호출되는 메서드라면 성능에 악영향을 끼칠 수 있다

## 해결법
### 1. 최소 인수 개수가 필요한 경우, “고정 매개변수 + varargs”
```java
static int min(int firstArg, int... restArgs) {
    int min = firstArg;
    for (int arg : restArgs) {
        if (arg < min) {
            min = arg;
        }
    }
    return min;
}
```
- 고정 매개변수로 최소 인수 개수를 강제할 수 있다
- 첫 번째 값으로 초기화해 null-check 불필요하다

### 2. 성능 문제 해결법
- 자주 호출되는 메서드라면, 가변인수 메서드 오버로딩 버전을 미리 만들어두자
- 가변인수 메서드는 내부적으로 배열을 생성하므로, 호출 빈도가 높은 메서드라면 성능에 영향을 줄 수 있다
- 위 예시처럼 0개, 1개, 2개, 3개 인수를 받는 오버로딩 메서드를 미리 만들어두면, 자주 호출되는 경우 성능 향상을 기대할 수 있다
```java
public void foo() { }                    // 배열 생성 X
public void foo(int a1) { }              // 배열 생성 X
public void foo(int a1, int a2) { }      // 배열 생성 X
public void foo(int a1, int a2, int a3) { } // 배열 생성 X
public void foo(int a1, int a2, int a3, int... rest) { } // 여기서만 배열 생성
```
- 이 성능 최적화 패턴은 **항상 적용해야 하는 규칙이 아니며**, API 복잡도가 올라가기 때문에 **가변인수 호출 비용이 실제 성능 병목이 되는 상황에서만 선택적으로 사용해야 한다.** 일반적인 상황에서는 단일 varargs 메서드만으로도 충분하다

## 추가
**EnumSet 정적페터리 of 메서드**도 가변인수를 사용하여 열거 타입 집합 생성 비용을 최소화한다
```java
public static <E extends Enum<E>> EnumSet<E> of(E e1)
public static <E extends Enum<E>> EnumSet<E> of(E e1, E e2)
public static <E extends Enum<E>> EnumSet<E> of(E e1, E e2, E e3)
public static <E extends Enum<E>> EnumSet<E> of(E e1, E e2, E e3, E e4)
public static <E extends Enum<E>> EnumSet<E> of(E e1, E e2, E e3, E e4, E e5)
public static <E extends Enum<E>> EnumSet<E> of(E first, E... rest) // 가변인수 버전
```
- 보통 EnumSet.of(…)는 대부분 1~5개 정도 요소를 넣고 쓰는 경우가 많기 때문에 가변인수 버전 외에 1~5개 요소를 받는 오버로딩 메서드를 미리 만들어두어 성능을 최적화했다

## QnA
Q. 가변인수 메서드를 사용할 때 주의할 점 + 해결 방법은?
