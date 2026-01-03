# item85. 자바 직렬화의 대안을 찾으라

Java 1.1 (1997) 부터 객체 직렬화(Serializable) 기능을 제공해왔습니다.

당시 자바의 핵심 목표는 "네트워크를 통해 객체를 그대로 주고받을 수 있는 언어" 였고, 객체를 메모리에서 그대로 보내고 그대로 복원하는 것이 개발의 편리성을 높인다고 생각하여 `Serializable` 인터페이스가 등장했습니다.

그러나 시간이 지나면서 여러 문제들이 드러나게 됩니다.

따라서 Effective Java에서는 “가능하다면 자바 직렬화를 사용하지 말고, 대안을 선택하라.” 라고 말합니다.

이유는 단순히 “불편해서”가 아니라, 보안·유지보수·확장성 측면에서 구조적인 문제가 많기 때문입니다.

## 1. 직렬화란?

직렬화는 객체를 바이트 스트림 형태로 변환하는 과정입니다.

```java
User user = new User("kim", 20);
```

이 객체는 메모리 상에만 존재하므로

- 파일에 저장하거나
- 네트워크로 전송하거나
- 캐시에 넣기 위해

"메모리 안의 객체 → 바이트 형태" 로 바꾸는 과정이 직렬화입니다.

반대로 바이트를 다시 객체로 복원하는 것을 역직렬화 라고 합니다.


## 2. Java의 직렬화

```java
class User implements Serializable {
    String name;
    int age;
}
```
```java
ObjectOutputStream out = new ObjectOutputStream(...);
out.writeObject(user);
```
```java
ObjectInputStream in = new ObjectInputStream(...);
User user = (User) in.readObject();
```

자바에서는 `Serializable` 인터페이스를 구현하면 직렬화가 가능합니다.

`ObjectOutputStream` / `ObjectInputStream` 을 통해 객체를 그대로 저장하고 복원할 수 있습니다.

## 3. Java 직렬화의 위험성

### 1) 생성자를 거치지 않는다

```java
ObjectInputStream.readObject();
```

자바 직렬화는 객체를 생성할 때 생성자를 호출하지 않습니다.

즉,

- 생성자에서 하던 유효성 검사
- 방어 코드
- 불변식 유지 로직

객체가 원래 지켜야 할 규을 모두 우회합니다.

공격자는 조작된 직렬화 데이터를 보내서 의도하지 않은 객체를 생성하게 만들 수 있습니다.

역직렬화는 신뢰할 수 없는 데이터에 대해 절대 사용하면 안됩니다.

### 2) 가젯 체인 문제

```java
ObjectInputStream.readObject()
  → SomeClass.readObject()
     → someMethod()
         → Runtime.getRuntime().exec("rm -rf /")
```

직렬화의 가장 치명적인 문제는 가젯 체인(Gadget Chain) 입니다.

> 가젯 메서드란 공격자가 직접 만든 게 아닌데도, 역직렬화 과정에서 자동으로 실행되는 메서드를 말합니다.
> 
> 가젯 체인이란, 역직렬화 과정에서 자동으로 호출되는 메서드들이 연결되어 의도하지 않은 동작을 실행하는 구조입니다.

즉, 공격자는

- 직접 코드를 실행하지 않아도
- 이미 클래스패스에 존재하는 클래스들만 조합해서
- 원격 코드 실행까지 유도할 수 있습니다.


### 3) 역직렬화 폭탄

```java
Set<Object> s1 = new HashSet<>();
Set<Object> s2 = new HashSet<>();

s1.add(s2);
s2.add(s1);
```

이 구조를 직렬화한 뒤 역직렬화하면

- equals(), hashCode()가 서로를 계속 호출
- 스택 오버플로우 또는 CPU 고갈

즉, 단순히 읽기만 해도 서버가 죽을 수 있습니다.

### 4) 클래스 구조 변경에 매우 취약

```java
class User implements Serializable {
    String name;
    int age;
}
```
```java
String email;
```

위 클래스에 필드 하나만 추가해도,

- 예전 직렬화 데이터와 호환이 깨짐
- `InvalidClassException` 발생
- `serialVersionUID` 관리 지옥 시작

이라는 문제들이 발생할 수 있습니다.

따라서 리팩토링이 곧 장애로 이어질 수 있습니다.

### 5) 언어 독립성 없음

Java 직렬화 포맷은

- Java 전용
- 다른 언어에서 해석 불가
- 장기 저장/버전 관리에 최악

반면 JSON / Protobuf 등의 데이터 포맷은

- 언어 독립적
- 구조가 명확
- 호환성 관리가 쉬움

## 4. 대안

### 1) JSON: DTO를 만들고 “데이터”만 직렬화/역직렬화

핵심은 “도메인 객체를 그대로 저장/전송”이 아니라, 전송/저장용 DTO를 정의하고 그 DTO를 JSON으로 변환하는 것이다.

#### DTO 정의 :

```java
public record UserDto(String name, int age) {}
```

#### HTTP 통신에서 JSON 사용 :

```java
ObjectMapper om = new ObjectMapper();

UserDto dto = new UserDto("kim", 20);
String json = om.writeValueAsString(dto);
// {"name":"kim","age":20}

httpClient.post("/users", json);
```

```java
String body = httpResponseBody; // {"name":"kim","age":20}
UserDto dto = om.readValue(body, UserDto.class);
```

#### Redis 캐시에서 JSON 사용 :

```java
// 저장
String json = om.writeValueAsString(new UserDto("kim", 20));
redis.set("user:1", json);

// 조회
String cached = redis.get("user:1");
UserDto dto = om.readValue(cached, UserDto.class);
```

ObjectInputStream / readObject() 로 객체 그래프를 복원하지 않습니다.

JSON은 값만 파싱하기 때문에 직렬화 취약점류와 거리가 멀어집니다.


### 2) Protobuf: 스키마(.proto) 기반 바이너리 포맷을 사용

Protobuf는 스키마 기반이라 호환성 관리가 쉽고, JSON보다 작고 빠릅니다.

(서비스 간 내부 통신, 이벤트 메시지, 고성능이 필요한 곳에서 자주 사용)

#### 스키마 정의 (user.proto)

```protobuf
syntax = "proto3";

package example;

message User {
  string name = 1;
  int32 age = 2;
}
```

#### 직렬화 (바이너리로)

```java
example.User user = example.User.newBuilder()
        .setName("kim")
        .setAge(20)
        .build();

byte[] bytes = user.toByteArray(); // 직렬화
```

#### 역직렬화 (바이너리에서 복원)

```java
example.User parsed = example.User.parseFrom(bytes);
System.out.println(parsed.getName()); // kim
```

Protobuf는 "자바 객체 그래프를 통째로 복원"하는 방식이 아닙니다.

스키마에 정의된 필드만 읽어서 메시지 객체를 구성합니다.

### 3) 화이트리스트(ObjectInputFilter)

자바 9부터는 직렬화의 치명적인 문제를 완화하기 위해 직렬화 필터(ObjectInputFilter) 라는 기능이 추가되었습니다.

이것은 어떤 클래스의 역직렬화를 허용할지를 명시적으로 제한하는 장치입니다.

```java
ObjectInputFilter filter = info -> {
    if (info.serialClass() != null &&
        info.serialClass().getName().startsWith("com.myapp.dto")) {
        return ObjectInputFilter.Status.ALLOWED;
    }
    return ObjectInputFilter.Status.REJECTED;
};

ObjectInputStream in = new ObjectInputStream(inputStream);
in.setObjectInputFilter(filter);
```

ObjectInputFilter 는 역직렬화 중에 클래스, 배열 크기, 객체 수, 깊이 등을 검사하고 조건에 맞지 않으면 즉시 차단하는 메커니즘입니다.

이렇게 하면,
- `com.myapp.dto` 패키지 외의 클래스는 역직렬화 불가
- 공격자가 JDK 내부 클래스나 가젯 체인을 주입해도 차단됨

그러나 화이트리스트는 차선택이지 근본적인 해결책은 아닙니다.

왜냐하면,

- 여전히 `readObject()`가 호출됨
- 객체 생성 메커니즘 자체는 위험
- 실수로 필터를 빠뜨리면 즉시 취약해짐
- 모든 라이브러리에서 완벽히 적용하기 어려움


## 5. 결론

> 자바 직렬화는 편하지만, 보안·유지보수·확장성 측면에서 위험한 선택입니다.
> 
> 따라서, 가능하면 자바 직렬화를 사용하지 말고 대안을 찾는 것이 좋습니다.


---

### 질문

Q. 자바 직렬화가 “위험하다”고 하는 이유는 무엇인가요?

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 85