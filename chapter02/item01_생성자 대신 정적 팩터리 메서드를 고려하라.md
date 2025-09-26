# ITEM01: 생성자 대신 정적 팩터리 메서드를 고려하라

**클래스의 인스턴스를 얻을 수 있는 방법**

1. public 생성자
```java
public class User {

    public User() {
    }
}
```

2. 정적 팩터리 메서드
```java
public class User {

    private User() {
    }

    public static User createUser() {
        return new User();
    }
}
```

## 정적 팩터리 메서드의 장점

### 1. 이름을 가질 수 있다.

- 생성자: 객체의 특성을 드러내기 어렵다
- **정적 팩터리 메서드: 이름을 통해 객체의 특성을 쉽게 파악 가능하다**

```java
public class User {
    private String name;
    private String phoneNumber;

    private User() {
    }

    // 생성자
    public User(String name) {
        this.name = name;
    }

    // 정적 팩터리 메서드
    public static User createUserWithName(String name) {
        User user = new User();
        user.name = name;
        return user;
    }
}


User user1 = new User("leevigong");                // 생성자: 이름이 없어 의도를 알기 어려움
User user2 = User.createUserWithName("leevigong"); // 정적 팩터리 메서드: 이름으로 의도를 명확히 알 수 있음
```

- 생성자: 시그니처가 같아 중복 불가하다
- **정적 팩터리 메서드: 같은 파라미터의 인스턴스 반환 객체 생성 가능하다**

```java
public class User {
    private String name;
    private String address;

    private User() {
    }

    // 생성자
    public User(String name) {
        this.name = name;
    }

    // 👎 생성자 오버로딩 불가: 두 생성자의 시그니처가 동일하여 컴파일 오류 발생 !
    // public User(String address) {
    //     this.address = address;
    // }

    // 정적 팩터리 메서드
    public static User createUserWithName(String name) {
        User user = new User();
        user.name = name;
        return user;
    }

    // 👍  제약 없이 같은 파라미터의 인스턴스 반환 객체 생성 가능
    public static User createUserWithAddress(String address) {
        User user = new User();
        user.address = address;
        return user;
    }
}
```

### 2. 호출될 때마다 인스턴스를 새로 생성하지 않아도 된다.

- 생성자: 호출될 때마다 새로운 인스턴스를 생성한다
- **정적 팩터리 메서드: 필요에 따라 인스턴스를 새로 생성하거나 재사용할 수 있다**
    - 특히, **불변 클래스(Immutable class)** 는 인스턴스를 캐싱하여 재활용함으로써 불필요한 객체 생성을 막고 성능을 높일 수 있다

```java
// 생성자
public class User {
    public User() {
    }
    
    public static void main(String[] var0) {
        User user1 = new User();
        User user2 = new User();
        System.out.println(user1 == user2); // false - 서로 다른 객체 참조
    }
}
```

```java
// 정적 팩터리 메서드
public class User {
    private static final User DEFAULT_USER = new User(); // 상수로 하나의 인스턴스 생성

    private User() { // private 생성자로 외부에서 인스턴스 생성 불가
    }

    public static User getInstance() {
        return DEFAULT_USER;
    }

    public static void main(String[] var0) {
        User user1 = getInstance();
        User user2 = getInstance();
        System.out.println(user1 == user2); // true - 동일한 객체 참조
    }
}
```

`DEFAULT_USER`이라는 **상수**를 만들어 두고, `getInstance()`가 처음 호출되면 객체를 생성하며, 이후에는 기존 객체를 반환한다  
➡ 매번 새로운 객체를 만들지 않고 하나의 인스턴스를 재사용한다(= 항상 동일한 객체만 사용되므로 인스턴스가 통제된다)


### 불변 클래스 (Immutable class)
- 한 번 생성되면 상태를 변경할 수 없는 클래스
- 대표 예시: `String`, `Boolean`, `Integer`, `Float`, `Long`

예시) `Boolean.valueOf()` <- 객체를 아예 생성하지 않는다
```java
public final class Boolean implements java.io.Serializable, Comparable<Boolean> {
    public static final Boolean TRUE = new Boolean(true);
    public static final Boolean FALSE = new Boolean(false);

    public static Boolean valueOf(boolean b) {
        return (b ? TRUE : FALSE);
    }
}

public static void main(String[] args) {
    Boolean boolean1 = Boolean.valueOf(true);
    Boolean boolean2 = Boolean.valueOf("true");
    Boolean boolean3 = new Boolean(true);

    System.out.println(boolean1 == boolean2); // true - 캐싱된 객체 재사용
    System.out.println(boolean1 == boolean3); // false - new는 다른 객체 생성
}
```

### 인스턴스 통제(instance-controlled) 클래스
\: 정적 팩터리 방식으로 인스턴스를 통제할 수 있는 클래스

- **싱글턴(singleton)** 보장
- 인스턴스화 불가(noninstantiable)로 만들 수 있음
- 불변 값 클래스에서 동치인 인스턴스가 단 하나뿐임을 보장 (a == b 일 때만 a.equals(b) 성립)
- **플라이웨이트(Flyweight) 패턴**의 근간
- 열거 타입은 인스턴스 하나만 생성함을 보장

> Flyweight 패턴  
> : **재사용 가능한 객체 인스턴스를 공유**시켜 **메모리 사용량을 최소화**하는 구조 패턴으로, 캐시(Cache) 개념을 디자인 패턴으로 일반화한 것
> - 변하지 않는 속성(intrinsic)은 캐싱하여 재사용
> - 자주 변하는 속성(extrinsic)은 별도로 분리하여 관리

> Singleton 패턴  
> : 특정 클래스가 단 하나만의 인스턴스를 생성하여 사용하기 위한 패턴
> ```java
> class Singleton {
>     private static Singleton singleton = null;
>
>     private Singleton() {}
>
>     static Singleton getInstance() {
>        if (singleton == null) {
>           singleton = new Singleton();
>      }
>     return singleton;
>   }
> }
> ```

### 3. 반환 타입의 하위 타입 객체를 반환할 수 있다.
리턴타입의 하위 타입인 인스턴스를 만들어줘도 되기 때문에, 리턴 타입은 인터페이스로 지정하고 구현 클래스를 API에 노출시키지 않고도 그 객체를 반환할 수 있어, API를 작게 유지할 수 있다  
이는 인터페이스를 정적 팩토리 메서드의 반환 타입으로 사용하는 인터페이스 기반 프레임워크를 만드는 핵심 기술이기도 하다
- 구현체를 외부에 드러내지 않고 인터페이스만 노출 가능
- 클라이언트는 반환된 객체의 실제 클래스를 몰라도 되고, 인터페이스만 사용하면 된다
- 구현체를 바꿔도 API는 수정 불필요

➡ 유연성과 확장성 ⬆
```java
public interface Animal {
    void sound();
}

class Dog implements Animal {
    public void sound() { System.out.println("멍멍"); }
}

class Cat implements Animal {
    public void sound() { System.out.println("야옹"); }
}

public class AnimalFactory {
    public static Animal create(String type) {
        if ("dog".equalsIgnoreCase(type)) return new Dog();
        if ("cat".equalsIgnoreCase(type)) return new Cat();
        throw new IllegalArgumentException("Unknown type");
    }
}

// 예시
Animal animal = AnimalFactory.create("dog"); // 실제로는 Dog 객체 반환
animal.sound(); // 멍멍
```

**실제 예**
- Collections.unmodifiableList(List<T>) → 실제로는 UnmodifiableList 하위 클래스 반환

<details>
<summary>참고</summary>

Java 8 이전
- 인터페이스에 정적 메서드 선언 불가 → "Type"인 인터페이스를 반환하기 위해선 "Types" 같은 동반 클래스(companion class) 필요
- 예: java.util.Collections → List, Set, Map 같은 인터페이스 관련 유틸 기능을 모아둔 클래스
    - 인터페이스대로 동작하는 객체를 얻을 것임을 알기에 굳이 별도 문서를 찾아가며 실제 구현 클래스가 무엇인지 알아보지 않아도 된다
    - 정적 팩토리 메서드를 사용하는 클라이언트는 얻은 객체를 (그 구현 클래스가 아닌) 인터페이스만으로 다루게 된다.

Java 8
- 인터페이스에 정적 메서드 선언 가능 → 동반 클래스 필요성 감소
- 단, 일부 구현은 여전히 별도의 package-private 클래스에 두어야 함 (인터페이스에는 public 정적 멤버만 허용하므로)

Java 9
- 인터페이스에 private 정적 메서드 선언 가능 → 코드 중복 제거에 도움
  ```java
    public interface StringUtils {
        static boolean isNullOrEmpty(String s) {
            return isNull(s) || s.isEmpty();
        }
    
        static boolean isBlank(String s) {
            return isNull(s) || s.trim().isEmpty();
        }
    
        // 인터페이스 내부 전용 헬퍼 메서드
        private static boolean isNull(String s) {
            return s == null;
        }
    }
    ```
- 단, private 정적 필드와 정적 멤버 클래스는 여전히 불가! 반드시 public
</details>


### 4. 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다.
**반환 타입의 하위 타입이기만 하면 어떤 클래스의 객체도 반환할 수 있다**
```java
public interface MyInt {
    static MyInt of(int v) {
        if (v > 100) return new MyBigInt(v);
        return new MySmallInt(v);
    }
    
    String getValue();

    class MySmallInt implements MyInt {
        private final int value;
        MySmallInt(int v) { this.value = v; }
        @Override public String getValue() { return "MySmallInt: " + value; }
    }

    class MyBigInt implements MyInt {
        private final int value;
        MyBigInt(int v) { this.value = v; }
        @Override public String getValue() { return "MyBigInt: " + value; }
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        MyInt small = MyInt.of(10);
        MyInt big = MyInt.of(200);

        System.out.println(small.getValue()); // MySmallInt: 10
        System.out.println(big.getValue());   // MyBigInt: 200
    }
}
```
- 클라이언트는 MyInt.of()만 알면 되고, 실제로 어떤 구현체가 반환되는지는 알 필요가 없다
- 내부적으로는 조건에 따라 다른 하위 클래스 인스턴스를 반환하지만, 클라이언트는 인터페이스만 다룬다
- 불필요해진 구현체는 자유롭게 제거하거나 새로운 구현체를 추가할 수도 있다

**실제 예**
- EnumSet.of(E...) → 원소 개수에 따라 RegularEnumSet 또는 JumboEnumSet


### 5. 정적 팩터리 메서드가 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.
이런 유연한 점을 이용하여 **서비스 제공자 프레임워크(Service Provider Framework)** 를 만드는 근간이 된다

서비스 제공자 프레임워크의 3가지의 핵심 컴포넌트
- 서비스 인터페이스(Service Interface): 구현체의 동작을 정의
- 제공자 등록 API(Provider Registration API): 제공자가 구현체를 등록할 때 사용
- 서비스 접근 API(Service Access API): 클라이언트가 서비스의 인스턴스를 얻을 때 사용 → **정적 팩터리 실체**
- \+ 서비스 제공자 인터페이스(Service Provider Interface): 서비스 인터페이스의 인스턴스를 생성하는 팩토리 객체를 설명

**대표적인 서비스 제공자 프레임워크 - JDBC(Java Database Connectivity)**  
\: MySql, OracleDB, MariaDB등 다양한 Database를 JDBC라는 프레임워크로 관리
- 서비스 인터페이스 = Connection
- 제공자 등록 API = DriverManager.registerDriver()
- 서비스 접근 API = DriverManager.getConnection()
- 서비스 제공자 인터페이스 = Driver

클라이언트가 `DriverManager.getConnection()`을 호출할 때마다 입력에 따라 서로 다른 구현체를 반환하지만, 클라이언트는 구체적인 클래스에 의존하지 않고`DatabaseConnection` 인터페이스만 사용한다는 점을 보여준다
```java
// 서비스 인터페이스
public interface DatabaseConnection {
    void connect();
}

// MySQL 구현체
class MySQLConnection implements DatabaseConnection {
    @Override
    public void connect() {
        System.out.println("MySQL 데이터베이스에 연결합니다.");
    }
}

// Oracle 구현체
class OracleConnection implements DatabaseConnection {
    @Override
    public void connect() {
        System.out.println("Oracle 데이터베이스에 연결합니다.");
    }
}

// 서비스 접근 API (정적 팩터리 메서드)
public class DriverManager {
    public static DatabaseConnection getConnection(String dbType) {
        if ("mysql".equalsIgnoreCase(dbType)) {
            return new MySQLConnection();
        } else if ("oracle".equalsIgnoreCase(dbType)) {
            return new OracleConnection();
        } else {
            throw new IllegalArgumentException("지원하지 않는 DB 타입입니다.");
        }
    }
}

// 사용 예:
public class Main {
    public static void main(String[] args) {
        DatabaseConnection conn1 = DriverManager.getConnection("mysql");
        conn1.connect();  // 출력: MySQL 데이터베이스에 연결합니다.

        DatabaseConnection conn2 = DriverManager.getConnection("oracle");
        conn2.connect();  // 출력: Oracle 데이터베이스에 연결합니다.
    }
}
```

**서비스 제공자 프레임 워크 패턴의 다양한 변형**
- 브리지(Bridge) 패턴
- 의존 객체 주입(Dependency Injection) 프레임워크

> [!NOTE]
> ⭐️ 장점 3, 4, 5 결론: 인터페이스나 추상 클래스를 반환 타입으로 지정하고, 내부 로직에 따라 서로 다른 구현체 인스턴스를 반환할 수 있다!

## 정적 팩터리 메서드의 단점

### 1. 상속을 하려면 public 또는 protected 생성자가 필요해 정적 팩터리 메서드만 제공하면 하위 클래스를 만들 수 없다.

상속을 하게되면 super() 를 호출하며 부모 클래스의 함수들을 호출한다. 그러나 부모 클래스의 생성자가 private이라면 상속이 불가능하다  
보통 정적 팩토리 메서드만 제공하는 경우 생성자를 통한 인스턴스 생성을 막는 경우가 많다

ex) Collections는 생성자가 private으로 구현되어 있기 때문에 상속할 수 없다.

이 제약은 상속보다 컴포지션을 사용하도록 유도하고, 불변 타입으로 만들려면 이 제약을 지켜야 한다는 점에서 오히려 장점으로 받아들이기도 한다  
** 컴포지션: 기존 클래스를 확장하는 대신에 새로운 클래스를 만들고 private 필드로 기존 클래스의 인스턴스를 참조하는 방식

### 2. 정적 팩터리 메서드는 프로그래머가 찾기 어렵다.
- 생성자는 클래스 이름과 동일한 이름을 가지므로 쉽게 찾을 수 있지만, 정적 팩터리 메서드는 이름이 다양하여 찾기 어려울 수 있다
- 정적 팩터리 메서드의 이름을 잘 짓는 것이 중요하다

#### 정적 팩터리 메서드 명명 방식
- from: **매개변수를 하나** 받아서 해당 타입의 인스턴스를 반환하는 형변환 메서드   
  `Date d = Date.from(instant);`
- of: **여러 매개변수**를 받아 적합한 타입의 인스턴스를 반환하는 집계 메서드  
  `Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);`
- instance 혹은 getInstance: 매개변수로 명시한 인스턴스를 반환, 같은 인스턴스임을 보장 X  
  `StackWalker luke = StackWalker.getInstance(options);`
- create 혹은 newInstance: instance 혹은 getInstance와 같지만, 매번 새로운 인스턴스 생성을 보장  
  `Object newArray = Array.newInstance(classObject, arrayLen);`
- getType: getInstance와 같으나, 생성할 클래스가 아닌 다른 클래스에 팩터리 메서드를 정의할 때 사용  
  `FileStore fs = Files.getFileStore(path); ("Type"은 팩터리 메서드가 반환할 객체의 타입)`
- newType: newInstance와 같으나, 생성할 클래스가 아닌 다른 클래스에 팩터리 메서드를 정의할 때 사용  
  `BufferedReader br = Files.newBufferedReader(path);`
- type: getType과 newType의 간결한 버전  
  `List<Complaint> litany = Collections.list(legacyLitany);`
## 정리
정적 팩터리 메서드와 public 생성자는 각자의 쓰임새가 있으니 상대적인 장단점을 이해하고 사용하는 것이 좋다.  
그렇다고 하더라도 정적 팩터리를 사용하는 게 유리한 경우가 더 많으므로 무작정 public 생성자를 제공하던 습관이 있다면 고치자.

---
## 질문
Q. 정적 팩터리 메서드의 장점 5가지는 무엇인가요?  
Q. 정적 팩터리 메서드의 단점 2가지는 무엇인가요?  
Q. 정적 팩터리 메서드의 명명 방식에는 어떤 것들이 있나요? (최소 2개 이상) 
