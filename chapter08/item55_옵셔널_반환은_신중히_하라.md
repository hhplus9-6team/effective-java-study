# item55. 옵셔널 반환은 신중히 하라.

## 왜 Optional이 등장했나?

자바 8 이전에는 메서드가 값을 반환할 수 없는 상황에서 두 가지 선택지밖에 없었습니다.

```java
// 방법 1: 예외 던지기
public static String findUserName(int id) {
    User user = database.find(id);
    if (user == null) {
        throw new IllegalArgumentException("유저 없음!");
    }
    return user.getName();
}

// 방법 2: null 반환
public static String findUserName(int id) {
    User user = database.find(id);
    if (user == null) {
        return null;  // 위험한 녀석...
    }
    return user.getName();
}
```

두 방법 모두 문제가 있습니다.

### 자바 8 이전의 딜레마

| 예외 던지기 | null 반환 |
|------------|-----------|
| 스택 추적 캡처 비용 큼 | 호출자가 null 체크 깜빡하면 |
| 진짜 예외 상황 아닌데 예외 쓰는 건 부적절 | 나중에 엉뚱한 곳에서 NullPointerException 터짐 |

---

## Optional 기본 사용법

Optional은 "값이 있을 수도 있고 없을 수도 있다"는 걸 타입으로 명시합니다.

```java
// Before: 예외 던지는 버전
public static <E extends Comparable<E>> E max(Collection<E> c) {
    if (c.isEmpty())
        throw new IllegalArgumentException("빈 컬렉션");
    
    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
            result = e;
    return result;
}

// After: Optional 반환 버전
public static <E extends Comparable<E>> Optional<E> max(Collection<E> c) {
    if (c.isEmpty())
        return Optional.empty();  // 빈 옵셔널 반환
    
    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
            result = e;
    return Optional.of(result);   // 값이 든 옵셔널 반환
}
```

스트림을 쓰면 더 간단해집니다.

```java
public static <E extends Comparable<E>> Optional<E> max(Collection<E> c) {
    return c.stream().max(Comparator.naturalOrder());
}
```

---

## Optional 생성 방법 3가지

| 메서드 | 설명 |
|--------|------|
| `Optional.empty()` | 빈 옵셔널 |
| `Optional.of(value)` | 값이 든 옵셔널 (value가 null이면 NPE 발생!) |
| `Optional.ofNullable(value)` | value가 null이면 빈 옵셔널, 아니면 값이 든 옵셔널 |

```java
// 주의! Optional.of()에 null 넣으면 바로 터짐
String name = null;
Optional.of(name);           // NullPointerException!
Optional.ofNullable(name);   // OK, 빈 Optional 반환
```

---

## Optional 값 꺼내는 방법들

실무에서 가장 많이 쓰는 패턴들입니다.

```java
Optional<String> opt = findUserName(userId);

// 1. 기본값 설정
String name = opt.orElse("손님");

// 2. 예외 던지기
String name = opt.orElseThrow(UserNotFoundException::new);

// 3. 확신이 있을 때 바로 꺼내기 (위험!)
String name = opt.get();  // 비어있으면 NoSuchElementException

// 4. 기본값 생성 비용이 클 때 (지연 생성)
String name = opt.orElseGet(() -> expensiveOperation());
```

흐름을 그림으로 보면 이렇습니다.

```
                    Optional<T>
                        │
           ┌────────────┴────────────┐
           │                         │
        값 있음                    값 없음
           │                         │
     ┌─────┴─────┐           ┌───────┴───────┐
     │           │           │               │
   get()    orElse(X)   orElse(X)      orElseThrow()
     │           │           │               │
     ▼           ▼           ▼               ▼
   값 T        값 T       기본값 X        예외 발생
```

---

## isPresent()는 신중히!

isPresent()로 값 존재 여부를 확인할 수 있지만, 대부분 더 나은 방법이 있습니다.

```java
// 안 좋은 예: isPresent() 남용
Optional<ProcessHandle> parent = ph.parent();
if (parent.isPresent()) {
    System.out.println("부모 PID: " + parent.get().pid());
} else {
    System.out.println("부모 PID: N/A");
}

// 좋은 예: map() 활용
String pid = ph.parent()
               .map(h -> String.valueOf(h.pid()))
               .orElse("N/A");
System.out.println("부모 PID: " + pid);
```

---

## Optional을 쓰면 안 되는 곳

### Optional 사용 금지 영역

| 금지 항목 | 이유 |
|-----------|------|
| 컬렉션/배열/스트림을 감싸지 말 것 | `Optional<List<T>>` → 그냥 빈 `List<T>` 반환 |
| Map의 값으로 쓰지 말 것 | `Map<K, Optional<V>>` → "키 없음" vs "빈 Optional" 혼란 |
| 컬렉션의 원소로 쓰지 말 것 | `List<Optional<T>>` → 복잡성만 증가 |
| 박싱된 기본 타입 담지 말 것 | `Optional<Integer>` → `OptionalInt` 사용 |

## 왜 컬렉션은 Optional로 감싸면 안될까?
```java
// 나쁜 예: Optional로 감싸기
public Optional<List<String>> findUserNames() {
List<String> names = repository.findNames();
if (names.isEmpty()) {
return Optional.empty();
}
return Optional.of(names);
}

// 좋은 예: 그냥 빈 리스트 반환
public List<String> findUserNames() {
List<String> names = repository.findNames();
if (names.isEmpty()) {
return Collections.emptyList();  // 빈 리스트 그대로
}
return names;
}
```
```java
// 빈 리스트면 그냥 루프 안 돔. 별도 체크 필요 없음
List<String> names = findUserNames();
for (String name : names) {
System.out.println(name);
}
```

---

## 핵심 차이를 그림으로 보면

### "없음"을 표현하는 방법 비교

**단일 값 (String, User 등)**

| 방법 | 평가 |
|------|------|
| null | 위험 (NPE 가능성) |
| Optional | 안전 ("없을 수 있음"을 타입으로 표현) |

**컬렉션 (List, Set, Map 등)**

| 방법 | 평가 |
|------|------|
| null | 위험 |
| Optional | 불필요한 중복! (컬렉션 자체가 빈 상태 표현 가능) |
| 빈 컬렉션 | 가장 좋음 |

---

## 비유로 설명하면

Optional은 **"선물 상자"** 라고 생각하면 됩니다.

- **단일 값(사과 1개)** 은 상자가 필요함
  - 사과가 있는지 없는지 상자 열어봐야 앎
- **컬렉션(바구니)** 은 이미 그 자체가 상자임
  - 바구니 자체가 비어있거나 뭔가 담겨있거나
  - 바구니를 또 상자에 넣을 필요 없음! (쓸데없이 복잡해짐)

```java
// 나쁜 예
public Optional<List<String>> getNames() {
    if (noData) return Optional.empty();
    return Optional.of(names);
}

// 좋은 예
public List<String> getNames() {
    if (noData) return Collections.emptyList();
    return names;
}
```

---

## 기본 타입 전용 Optional

박싱/언박싱 오버헤드를 피하기 위해 기본 타입 전용 클래스가 있습니다.

```java
// 나쁜 예: 두 겹으로 감쌈
Optional<Integer> count = Optional.of(100);  // int → Integer → Optional

// 좋은 예: 전용 클래스 사용
OptionalInt count = OptionalInt.of(100);     // 한 겹만
OptionalLong bigNumber = OptionalLong.of(1_000_000L);
OptionalDouble ratio = OptionalDouble.of(3.14);
```

---

## 언제 Optional을 반환해야 하나?

### Optional 반환 결정 기준

**YES, Optional 반환:**
- 결과가 없을 수 있고
- 클라이언트가 이 상황을 반드시 처리해야 할 때

**NO, Optional 말고 다른 것:**

| 상황 | 대안 |
|------|------|
| 성능이 극도로 중요한 경우 | null 또는 예외 |
| 컬렉션 반환 | 빈 컬렉션 |
| 기본 타입 | `OptionalInt` 등 전용 클래스 |

---

## 핵심 요약

Optional은 "값이 없을 수도 있다"는 의도를 명확히 전달하는 도구입니다. 하지만 만능이 아니고, 성능 오버헤드가 있으며, 컬렉션이나 Map의 값으로는 쓰면 안 됩니다. 반환 타입으로만 제한적으로 사용하는 게 좋습니다.

---

## 이해도 테스트

**문제 1.** 다음 중 Optional 생성 방법에 대한 설명으로 올바른 것은?

A) Optional.of(null)은 빈 Optional을 반환한다  
B) Optional.ofNullable(null)은 NullPointerException을 던진다  
C) Optional.empty()는 null을 담은 Optional을 반환한다  
D) Optional.of(value)에 null을 넣으면 NullPointerException이 발생한다

**문제 2.** Optional을 사용하면 안 되는 상황으로 적절한 것은?

A) 메서드가 값을 반환하지 못할 수도 있을 때  
B) Map의 값(value)으로 저장할 때  
C) 검색 결과가 없을 수 있는 API를 설계할 때  
D) 클라이언트가 빈 결과를 처리해야 할 때

**문제 3.** 다음 코드의 문제점은?

```java
public Optional<List<String>> findAllNames() {
    List<String> names = repository.findNames();
    if (names.isEmpty()) {
        return Optional.empty();
    }
    return Optional.of(names);
}
```

A) Optional.of() 대신 Optional.ofNullable()을 써야 한다  
B) 컬렉션을 Optional로 감싸면 안 된다. 빈 리스트를 반환해야 한다  
C) isEmpty() 체크가 불필요하다  
D) 제네릭 타입 선언이 잘못되었다

