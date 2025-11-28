# item 33. 타입 안전 이종 컨테이너를 고려하라

- **타입 안전(type-safe)**: 잘못된 타입의 값을 넣고 꺼낼 때 컴파일러가 잡아주거나, 최소한 런타임 이전에 오류가 드러나는 특성.
- **이종(heterogeneous)**: 서로 다른 종류의 값이 뒤섞여 있다는 뜻. “이중”이나 “두 가지”가 아니라, 여러 타입이 공존한다는 의미다.
- **컨테이너(container)**: 값들을 담아두는 자료구조 전반을 가리키는 표현. 컬렉션뿐 아니라 맵, 설정 저장소, 캐시 등도 포함한다.
- 따라서 **타입 안전 이종 컨테이너**란 “여러 타입의 값을 담되 타입 안전성을 유지하는 컨테이너”다.

## 한 컨테이너에 여러 타입을 담고 싶을 때
- 제네릭 컬렉션(`List<String>`, `Set<Integer>`)은 **오직 한 가지 타입만** 다루도록 설계되어 있다. 예를 들어 학번을 저장하는 `Set<Integer>`나 이름-나이쌍을 저장하는 `Map<String, Integer>`는 각각의 타입이 고정된다.
- 실제 애플리케이션에서는 “사용자 선호값”, “서비스 객체”, “컴포넌트 설정”, “데이터베이스 컬럼 메타데이터”처럼 **타입이 제각각인 데이터 묶음**을 한 곳에 저장하고 싶을 때가 많다.
- 이런 요구를 `Map<String, Object>`처럼 단순하게 처리하면, 꺼낼 때 매번 캐스팅해야 하고 실수도 잦다. 컴파일러가 타입 오류를 잡아줄 수도 없다.
- 아이템 33은 “컨테이너가 아니라 **키에 제네릭을 적용**하면 여러 타입을 안전하게 넣고 뺄 수 있다”는 해법을 제시한다.

## 타입 토큰(Type Token)
- 책에서는 타입 토큰을 “컴파일타임 타입 정보와 런타임 타입 정보를 알아내기 위해 메서드들이 주고받는 class 리터럴”이라 규정한다.
- `String.class`는 `Class<String>`, `Integer.class`는 `Class<Integer>`다. 이렇게 `Class<T>` 형태의 리터럴이 바로 타입 토큰이다.
- 이 토큰을 키로 사용하면, 제네릭 타입 시스템이 “값의 타입이 키와 같다”는 사실을 보장해 준다.

##  Favorites 예시
```java
public final class Favorites {
    private final Map<Class<?>, Object> favorites = new HashMap<>();

    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), instance);
    }

    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
```

```java
Favorites f = new Favorites();
f.putFavorite(String.class, "java");
f.putFavorite(Integer.class, 0xcafebabe);
f.putFavorite(Class.class, Favorites.class);

String favoriteString = f.getFavorite(String.class);
int favoriteInteger = f.getFavorite(Integer.class);
Class<?> favoriteClass = f.getFavorite(Class.class);

System.out.printf("%s %x %s%n", favoriteString, favoriteInteger, favoriteClass.getName());
```

### 어떻게 타입 안전성이 유지될까?
1. `putFavorite`은 `Class<T>`와 제네릭 타입 매개변수 `T`를 함께 받기 때문에, 키와 값의 타입을 **컴파일 타임에 묶어서** 저장한다.
2. 내부 맵은 `Map<Class<?>, Object>`지만, `getFavorite`은 `Class.cast`를 사용해 **동일한 타입 토큰으로 꺼낸 값만 허용**한다.
3. 따라서 `String`을 요청했는데 `Integer`를 돌려주는 상황이 발생하지 않는다.

### 왜 `Map<Class<?>, Object>`인가?
- 키로 어떤 클래스가 들어올지 모르기 때문에 `Class<?>`로 선언한다.
- 값 타입을 `Object`로 선언해둬야 모든 타입의 인스턴스를 저장할 수 있다.
- 대신 `getFavorite`에서 `type.cast(...)`로 검증해, 키와 값 사이의 타입 관계를 되살린다.

#### 와일드카드 타입, 제대로 이해하기
- `Class<?>`의 `?`는 **비한정적 와일드카드 타입(unbounded wildcard)**이다. “어떤 타입이든 상관없다”라는 의미이지만, 여기서는 **맵 전체가 아니라 키 쪽**에만 적용됐다.
- `Map<Class<?>, Object>`를 보고 “와일드카드가 있으니 아무것도 넣지 못한다”고 오해할 수 있지만, 실상은 그 반대다. 맵 타입 인자 2개 중 **키 타입 자리**(`Class<?>`)에만 와일드카드가 들어있다. 즉, `Class<String>`, `Class<Integer>`, `Class<Favorites>` 등 어떤 `Class` 리터럴이든 키로 쓸 수 있다.
- 책에서 말한 “와일드카드 타입이 중첩되었다”는 표현은, 외부 제네릭(`Map<...>`) 안쪽에 또 다른 제네릭(`Class<?>`)이 들어있는 구조를 가리킨다. 겉으로 보기엔 `Map` 자체가 와일드카드처럼 보이지만, 사실은 `Map`의 타입 매개변수 가운데 하나가 와일드카드인 셈이다.
- 그래서 `favorites.put(String.class, "hi")`처럼 키와 값 쌍을 자유롭게 저장할 수 있고, 대신 `Class.cast`를 통해 꺼낼 때 타입 일치를 다시 점검한다.
- 복습으로 와일드카드에는 `?`(비한정), `? extends T`(상한 경계), `? super T`(하한 경계)가 있다. 아이템 33에서는 “모든 클래스 키를 받고 싶다”는 요구에 맞춰 비한정 와일드카드를 사용한 것이다.



## 안전하지 않은 사용을 막는 방법
- `Class`의 raw 타입을 사용하면 다음과 같은 실수가 가능하다.
  ```java
  Favorites favorites = new Favorites();
  Class rawType = (Class)String.class; // raw 타입으로 취급
  favorites.putFavorite(rawType, 123); // 경고 발생, 런타임에 ClassCastException 위험
  ```
- 책에서는 `putFavorite`에서 들어온 값이 정말 그 타입의 인스턴스인지 검증하거나, raw 타입 사용을 금지하도록 문서화하라고 조언한다.
- `putFavorite`은 `Class<T>`와 `T`를 함께 받지만, 내부 맵에 저장되면서 값은 `Object`로 승격돼 **타입 링크(type linkage)** 정보가 사라진다. `getFavorite`이 같은 타입 토큰으로 검색해 `Class.cast`를 호출할 때 그 연결 정보가 다시 살아난다.
- `type.cast(instance)`는 잘못된 값이 들어올 경우 즉시 `ClassCastException`을 던지므로, 문제를 조기에 발견할 수 있다.
- 이 아이디어는 `Collections.checkedList`, `Collections.checkedSet`, `Collections.checkedMap`에도 적용돼 있다. 컬렉션과 함께 `Class` 객체를 받아, 런타임에 잘못된 타입이 들어오는 것을 감시하는 **실체화 뷰(reified view)**를 제공한다.

```java
List<String> strings = new ArrayList<>();
List rawList = strings;              // 경고 발생
rawList.add(42);                     // 컴파일은 되지만, strings에는 잘못된 값이 들어감

List<String> checked = Collections.checkedList(strings, String.class);
((List) checked).add(42);            // 즉시 ClassCastException
```

## 비실체화 타입(Non-Reifiable Type) 다루기
- `List<String>`처럼 런타임에 타입 정보가 지워지는 타입은 `Class<List<String>>` 형태로 표현할 수 없다.
- 이런 경우에는 **슈퍼 타입 토큰(TypeToken)** 또는 `ParameterizedTypeReference`와 같은 별도 키 클래스를 활용한다.
- 예시 아이디어:
  ```java
  public class TypeRef<T> {
      private final Type type;
      protected TypeRef() {
          this.type = ((ParameterizedType) getClass().getGenericSuperclass()).getActualTypeArguments()[0];
      }
      public Type getType() { return type; }
  }

  Favorites favorites = new Favorites();
  TypeRef<List<String>> stringList = new TypeRef<List<String>>() {};
  favorites.putFavorite(stringList, List.of("a", "b"));
  ```
- 책에서도 “비실체화 타입을 지원하려면 타입 토큰을 확장하거나 직접 만든 키 클래스를 사용하라”고 조언한다.
- 완벽한 해결책은 아니어서, 복잡도가 높아지고 런타임 비용도 늘 수 있다. 아이템 33은 “비실체화 타입에는 만족스러운 우회로가 아직 없다”고 솔직하게 언급한다.

## 한정적 타입 토큰과 어노테이션 API
- 모든 `Class` 객체를 허용하기보다, 특정 상위 타입을 구현한 클래스만 받도록 **한정(bound)된 타입 토큰**을 사용할 수 있다.
- 대표 예시는 어노테이션 리플렉션 API다.

```java
public interface Annotation {
    Class<? extends Annotation> annotationType();
}

public interface AnnotatedElement {
    <T extends Annotation> T getAnnotation(Class<T> annotationType);
}
```
- `getAnnotation`의 매개변수는 `Class<T extends Annotation>`이다. 어노테이션 타입만 허용하고, 그 결과를 안전하게 돌려준다.
- 정적 타입을 알 수 없을 때는 `Class.asSubclass`를 활용해 런타임에 안전한 형변환을 수행한다.

```java
static Annotation getAnnotation(AnnotatedElement element, String typeName) {
    try {
        Class<?> annotationType = Class.forName(typeName);
        return element.getAnnotation(annotationType.asSubclass(Annotation.class));
    } catch (ClassNotFoundException ex) {
        throw new IllegalArgumentException(ex);
    }
}
```
- `asSubclass`는 “이 클래스가 주어진 타입의 하위 타입인지”를 검사하고, 맞다면 `Class<? extends U>` 형태로 돌려준다. 아니면 `ClassCastException`을 즉시 던져 잘못된 사용을 막는다.

## 질문
Q1. Favorites 클래스에서 내부 맵이 Map<Class<?>, Object>로 선언되어 있는데도 getFavorite 메서드가 타입 안전성을 보장할 수 있는 이유는 무엇인가요?

Q2. 어노테이션 API의 `getAnnotation` 메서드에서 사용하는 한정적 타입 토큰은 어떻게 동작하며, 정적 타입을 알 수 없을 때 안전하게 형변환하는 방법은 무엇인가요?
