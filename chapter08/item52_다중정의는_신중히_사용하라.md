# item52. 다중정의는 신중히 사용하라

### 1. 다중정의(Overloading) 문제의 대표 사례

#### 예제 코드
```
static class CollectionClassifier {
    public static String classify(Set<?> s) {
        return "집합";
    }

    public static String classify(List<?> s) {
        return "리스트";
    }

    public static String classify(Collection<?> s) {
        return "그 외 컬렉션";
    }
}
```
#### 실행 결과
```
Collection<?>[] collections = {
    new HashSet<>(),
    new ArrayList<>(),
    new HashMap<>().values()
};

for (Collection<?> c : collections) {
    System.out.println(CollectionClassifier.classify(c));
}
```
```
그 외 컬렉션
그 외 컬렉션
그 외 컬렉션
```
오버로딩은 컴파일타임에 선택되며, 컴파일러는 변수의 선언타입만 보고 어떤 메서드를 선택할지 결정한다. 
이때, `collection`의 타입은 항상 `Collections<?>`이다.

### 2. 오버라이딩(override)은 문제 없음
#### 예제 코드
```
static class Wine { String name() { return "포도주"; } }
static class SparklingWine extends Wine {
    @Override String name() { return "발포성 포도주"; }
}
static class Champagne extends SparklingWine {
    @Override String name() { return "샴페인"; }
}
```
#### 실행 결과
```
List<Wine> list = List.of(new Wine(), new SparklingWine(), new Champagne());
for (Wine w : list) System.out.println(w.name());
```
```
포도주
발포성 포도주
샴페인
```
오버라이딩은 다형성 규칙을 따르고 런타임 실제 객체 타입을 기준으로 동적 디스패치가 일어난다.

### 3. 문제 해결 방법 - 오버로딩 제거하기
`instanceof` + 단일 메서드 조합
```
static class CollectionClassifier2 {
    public static String classify(Collection<?> s) {
        if (s instanceof Set) return "집합";
        else if (s instanceof List) return "리스트";
        return "그 외 컬렉션";
    }
}
```
### 다중 정의에서 주의할 점
- 다중 정의가 혼동을 일으키는 상황을 피하자
- 매개변수 수가 같은 다중 정의는 만들지 말자.
  - 혹여나 가변인수를 사용한다면, 다중 정의 자체를 만들지 말자.
- 다중 정의 대신 메서드 이름을 다르게 짓자.
  - `ObjectOutputStream` 의 경우 `writeBoolean()`, `writeInt()`와 같은 이름의 메서드를 제공한다.
  - 생성자의 경우엔 1개 이상이라면, 무조건 다중정의가 되어버린다. 이 경우엔 정적 팩터리라는 대안을 이용하여 생성 의도와 코드를 명확하게 할 수 있다.

### 4-1. 다중정의의 함점 : 오토박싱
#### 예제 코드
```
remove(int index)
remove(Object o)
```
```
List<Integer> list = new ArrayList<>();
Set<Integer> set = new TreeSet<>();

for (int i = -3; i < 3; i++) { set.add(i); list.add(i); }

for (int i = 0; i < 3; i++) {
    set.remove(i);          // remove(Object) 호출
    list.remove(i);         // remove(int index) 호출
}
```
#### 결과
```
set  = [-3, -2, -1]
list = [-2, 0, 2] //리스트는 인덱스 제거가 됨.
```
#### 해결(명시적 캐스팅)
```
list.remove((Integer) i));
```

### 다중 정의의 함정 : 람다와 메서드 참조
```
@Test
public void lambdaTest1() {
    new Thread(System.out::println).start();
}

@Test
public void lambdaTest2() {
    ExecutorService exec = Executors.newCachedThreadPool();
    exec.submit(System.out::println);
}
```
- lambdaTest1()의 경우 Thread()에서 함수형 인터페이스인 Runnable을 받기 때문에, 위와 같이 메서드 참조를 인수로 주었다.
- lambdaTest2()의 경우 exec.submit()에서도 함수형 인터페이스인 Runnable을 받는데, 또 다른 함수형 인터페이스인 Callable도 받는다. 그래서 컴파일러는 어떤 것을 선택해야 할 지 알 수 없어진다.
    ```
    submit(Runnable task)    
    submit(Callable<T> task)
    ```
    - println은 반환값이 없지만, 자바는 void-returning method ref를 Callable로도 억지로 걸어보려고 하면서 모호해짐.
- 즉, 서로 다른 함수형 인터페이스라도 같은 위치의 인수로 받아서는 안 된다.
#### 해결 (인수 포워드하기)
```
public boolean contentEquals(StringBuffer sb) {
    return contentEquals((CharSequence)sb);
}
```
명시적 캐스팅으로 어떤 메서드를 선택해야 하는지 강제한다.

### String 클래스의 다중 정의 오류
```
public static String valueOf(Object obj) {
    return (obj == null) ? "null" : obj.toString();
}

public static String valueOf(char data[]) {
    return new String(data);
}
```
- 같은 객체를 넘기더라도 전혀 다른 일을 수행한다.
- 이렇게할 이유가 없었으므로, 잘못 설계된 사례로 남아있다.

