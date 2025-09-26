# 싱글턴 패턴 구현 방법

## 싱글턴이란?
인스턴스를 오직 하나만 생성할 수 있는 클래스를 말합니다. 함수와 같은 무상태 객체나 설계상 유일해야 하는 시스템 컴포넌트가 대표적인 예입니다.

## 싱글턴 구현의 3가지 방법

### 1. public static final 필드 방식

```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() {...}
    public void leaveTheBuilding() {...}
}
```

**장점:**
- API에 싱글턴임이 명백히 드러남
- 간결함

**주의사항:** 리플렉션 공격 가능 (AccessibleObject.setAccessible 사용)

### 2. 정적 팩토리 메서드 방식

```java
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() {...}
    public static Elvis getInstance() { return INSTANCE; }
    public void leaveTheBuilding() {...}
}
```

**장점:**
- API 변경 없이 싱글턴이 아니게 변경 가능
- 제네릭 싱글턴 팩토리로 만들 수 있음
- 메서드 참조를 공급자로 사용 가능 (`Elvis::getInstance`)

### 3. 열거 타입 방식 (권장)

```java
public enum Elvis {
    INSTANCE;
    public void leaveTheBuilding() {...}
}
```

**장점:**
- 가장 간결함
- 직렬화 자동 처리
- 리플렉션 공격에 완벽히 안전함

**제약사항:** Enum 외의 클래스 상속 불가

## 직렬화 시 주의사항

싱글턴 클래스를 직렬화할 때는 `readResolve` 메서드를 추가해야 합니다:

```java
private Object readResolve() {
    return INSTANCE;
}
```

## 추가설명 
Q1. 정적 팩토리 메서드 방식에서는 메서드의 **구현부만 변경**하면 되기 때문에 API 변경 없이 싱글턴이 아니게 만들 수 있는 예시는?


### API 변경 없이 싱글턴이 아니게 변경

#### 1. 스레드별로 다른 인스턴스 제공
```java
public class Elvis {
    private static final ThreadLocal<Elvis> INSTANCES =
        ThreadLocal.withInitial(Elvis::new);

    private Elvis() {...}

    public static Elvis getInstance() {  // API는 동일!
        return INSTANCES.get();  // 스레드마다 다른 인스턴스
    }
}
```

#### 2. 매번 새로운 인스턴스 생성
```java
public class Elvis {
    private Elvis() {...}

    public static Elvis getInstance() {  // API는 동일!
        return new Elvis();  // 매번 새 인스턴스 생성
    }
}
```

#### 3. 풀(Pool)에서 인스턴스 관리
```java
public class Elvis {
    private static final List<Elvis> POOL = Arrays.asList(
        new Elvis(), new Elvis(), new Elvis()
    );
    private static int index = 0;

    private Elvis() {...}

    public static Elvis getInstance() {  // API는 동일!
        return POOL.get((index++) % POOL.size());  // 순환하며 반환
    }
}
```



public final 필드 방식에서는 `INSTANCE` 필드 자체가 final이므로 다른 객체를 참조하도록 변경할 수 없습니다. 싱글턴이 아니게 만들려면 클래스 전체를 수정해야 하므로 **API 변경**이 필요합니다.

**핵심**: 정적 팩토리 메서드는 **메서드 호출**을 통해 인스턴스에 접근하므로, 메서드 내부 로직만 바꾸면 클라이언트 코드는 그대로 두고도 동작을 변경할 수 있습니다.

Q2. 제네릭 싱글턴 팩토리는 **타입 안전성을 보장하면서 하나의 객체를 여러 타입으로 활용**할 수 있게 해주는 패턴이라는 예시는?


## 일반적인 예시: Collections.emptySet()

```java
// 제네릭 싱글턴 팩토리의 대표적인 예
Set<String> stringSet = Collections.emptySet();
Set<Integer> intSet = Collections.emptySet();
Set<Long> longSet = Collections.emptySet();
```

실제로는 모두 **같은 객체**를 참조하지만, 컴파일 타임에는 각각 다른 타입으로 처리됩니다.

## 직접 구현해보기

### 1. 기본 구조
```java
public class GenericSingleton<T> {
    // 실제 인스턴스는 하나만 존재
    private static final GenericSingleton<?> INSTANCE = new GenericSingleton<>();

    private GenericSingleton() {}

    // 제네릭 싱글턴 팩토리 메서드
    @SuppressWarnings("unchecked")
    public static <T> GenericSingleton<T> getInstance() {
        return (GenericSingleton<T>) INSTANCE;
    }

    public void doSomething(T item) {
        System.out.println("Processing: " + item);
    }
}
```

### 2. 사용 예시
```java
public class Main {
    public static void main(String[] args) {
        // 모두 같은 객체지만 타입이 다르게 처리됨
        GenericSingleton<String> stringSingleton = GenericSingleton.getInstance();
        GenericSingleton<Integer> intSingleton = GenericSingleton.getInstance();
        GenericSingleton<List<String>> listSingleton = GenericSingleton.getInstance();

        // 객체 동일성 확인
        System.out.println(stringSingleton == intSingleton); // true

        // 하지만 타입 안전성은 보장됨
        stringSingleton.doSomething("Hello");     // OK
        intSingleton.doSomething(42);             // OK
        // stringSingleton.doSomething(42);       // 컴파일 에러!
    }
}
```

## 실용적인 예시: Identity Function

```java
public class IdentityFunction<T> implements Function<T, T> {
    private static final IdentityFunction<?> INSTANCE = new IdentityFunction<>();

    private IdentityFunction() {}

    @SuppressWarnings("unchecked")
    public static <T> IdentityFunction<T> getInstance() {
        return (IdentityFunction<T>) INSTANCE;
    }

    @Override
    public T apply(T t) {
        return t; // 입력값을 그대로 반환
    }
}

// 사용
Function<String, String> stringIdentity = IdentityFunction.getInstance();
Function<Integer, Integer> intIdentity = IdentityFunction.getInstance();

System.out.println(stringIdentity.apply("test")); // "test"
System.out.println(intIdentity.apply(100));       // 100
```

## 왜 정적 팩토리에서만 가능한가?

### public final 필드 방식의 한계
```java
public class BadExample<T> {
    // 이렇게 할 수 없음! 컴파일 에러
    public static final BadExample<T> INSTANCE = new BadExample<>();
    // Static 필드에서는 클래스의 타입 매개변수 사용 불가
}
```

### 정적 팩토리 방식의 장점
```java
public class GoodExample<T> {
    private static final GoodExample<?> INSTANCE = new GoodExample<>();

    // 메서드 레벨에서 타입 매개변수 선언 가능
    public static <T> GoodExample<T> getInstance() {
        return (GoodExample<T>) INSTANCE;
    }
}
```

**핵심**: 정적 팩토리 메서드는 **메서드 레벨에서 제네릭**을 사용할 수 있기 때문에, 하나의 인스턴스를 여러 타입으로 안전하게 캐스팅해서 반환할 수 있습니다. 이는 메모리 효율성과 타입 안전성을 동시에 제공합니다.


## 예상 질문.

**다음 중 싱글턴 패턴의 세 가지 구현 방식에 대한 설명으로 올바르지 않은 것은?**

1. public static final 필드 방식은 API에 싱글턴임이 명백히 드러나는 장점이 있다.

2. 정적 팩토리 메서드 방식은 API 변경 없이 싱글턴이 아니게 변경할 수 있는 유연성을 제공한다.

3. 열거 타입 방식은 직렬화와 리플렉션 공격에 대해 완벽한 보호를 제공한다.

4. 열거 타입 방식은 다른 클래스를 상속받아야 하는 싱글턴에서도 사용할 수 있다.

5. 싱글턴 클래스를 직렬화할 때는 readResolve 메서드를 구현해야 새로운 인스턴스 생성을 방지할 수 있다.
