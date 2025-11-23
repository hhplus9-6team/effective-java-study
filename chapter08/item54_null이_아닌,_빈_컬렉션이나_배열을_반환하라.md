# item54. null이 아닌, 빈 컬렉션이나 배열을 반환하라

## 1. null을 반환하면 안 되는 이유

```java
List<String> getUsers() {
    if (noUserFound) return null;
    return users;
}
```

- 과거 자바 코드에서는 조회 결과가 없으면 null을 반환하는 방식이 흔했습니다.
- 그러나 null 반환은 API 안정성, 가독성, 일관성을 해치는 방법이었습니다.

### 1) 불필요한 null 체크 반복

```java
List<String> users = getUsers();
if (users != null) {
    for (String u : users) ...
}

```

- null 반환을 하게 되면 호출자마다 null 체크가 강제됩니다.
- 이로 인해, 코드가 지저분해지고 유지보수 비용이 증가합니다.

### 2) null 체크 하지 않으면 NPE 발생 

```java
for (User u : getUsers()) { ... } // ❌ getUsers()가 null이면 바로 NPE

getUsers().size(); // ❌ NPE
```
- null 체크를 하지 않으면 런타임 시점에 NullPointerException이 발생할 수 있습니다.

### 3) null 의미 모호성

- null이 의미하는 것이 모호하여 API 사용자가 매번 이를 해석해야 합니다.
  - 데이터 없음을 의미하는지? 
  - 초기화 전을 의미하는지? 
  - 오류 발생을 의미하는지?

### 4) API 일관성 깨짐

- 어떤 메서드는 null 반환하고 어떤 메서드는 빈 리스트 반환하면 예측이 어려워집니다. 
- 이는 API 일관성이 떨어져 사용자 입장에서 신뢰하기 어렵습니다.


## 2. null 대신 빈 컬렉션 / 빈 배열 반환하기

- 반복문에서 바로 사용이 가능합니다.
- NPE가 발생하지 않습니다.
- 호출자 코드가 간결해집니다.
- API 일관성이 확보됩니다.
- 빈 리스트는 재사용되므로 성능/메모리 오버헤드가 거의 없습니다.

### 1) 빈 컬렉션 반환

```java
public List<User> getUsers() {
    return users != null ? users : Collections.emptyList();
}
```

### 2) 필드를 빈 리스트로 초기화

```java
private final List<User> users = new ArrayList<>();

public Lisst<User> getUsers() {
    return users; 
}
```

### 3) 빈 배열 반환

```java
private static final String[] EMPTY_STRING_ARRAY = {};

String[] getNames() {
    return names != null ? names : EMPTY_STRING_ARRAY;
}
```

### 4) toArray(new String[0])

```java
return list.toArray(new String[0]);
```

- JVM이 내부적으로 정확한 크기의 배열을 최적화해서 생성
- `new String[list.size()]`를 미리 만들어 넘기는 것보다 빠르거나 동일함
- 배열을 미리 할당한다고 성능이 좋아지지 않음
  - 미리 큰 배열을 만들어 넘기면 배열을 초기화하고, 내부적으로 복사/채우기 작업이 들어가면 성능이 오히려 안 좋아질 수 있음
- 실수를 줄이고 관용적이라 가독성이 좋음

## 3. `Collections.emptyList()` vs `List.of()` vs `new ArrayList<>()` 

### 1) `Collections.emptyList()`

```java
List<String> list = Collections.emptyList();
```

- 길이 0, 불변 리스트
- 내부에 재사용되는 단일 인스턴스(싱글턴)라서 객체 하나만 계속 씀
- add/remove 하면 바로 `UnsupportedOperationException`
- 아무것도 없으며, 절대 수정되면 안되는 리스트일 때 사용하면 좋음

### 2) `List.of()`

```java
List<String> empty = List.of();            // 빈 불변 리스트
List<String> one   = List.of("a");        // 원소 1개
List<String> many  = List.of("a", "b");   // 원소 여러 개
```

- 자바 9+의 간편 불변 리스트 팩토리
- `Collections.emptyList()`와 비슷하게 불변
- 문법이 짧아서 하드코딩된 상수 리스트 표현할 때 매우 깔끔

### 3) `new ArrayList<>()`

```java
List<String> list = new ArrayList<>();
```

- 길이 0, 가변 리스트
- 호출할 때마다 새로운 객체 생성
- add/remove 마음대로 가능
- 지금은 비어있지만 나중에 안에서/밖에서 add할 리스트


## 4. 빈 컬렉션 / 배열 반환 시 성능 문제

- `Collections.emptyList()`는 싱글턴 객체로 매번 새로 생성되지 않습니다.
- 배열도 `static final` 로 캐싱해놓으면 비용이 들지 않습니다.
- 리스트 길이가 0인 경우 반복문은 0번 실행되므로 오버헤드도 미미하게 발생합니다.
- 따라서, 미세한 비용에 집착해 null 을 반환하는 건 오히려 유지보수성이나 안정성에서 손해를 초래할 수 있습니다.


## 5. 결론

- null을 반환하지 마라.

- 항상 빈 컬렉션 또는 빈 배열을 반환하라.

- API 사용자에게 불필요한 null 체크를 강요하지 말고, 예측 가능한 동작을 보장해라.

- 빈 컬렉션은 비용이 거의 없으며, 유지보수성과 안전성이 압도적으로 좋아진다.

---

### 질문

Q. null 대신 빈 컬렉션을 반환하는 것이 좋은 이유는 무엇인가?

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 54