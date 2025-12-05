# item64. 객체는 인터페이스를 사용해 참조하라

## 1. 핵심 개념
변수, 필드, 매개변수, 반환 타입을 구현 클래스가 아니라 인터페이스로 선언하라.
```
// 좋은 예
Set<Son> sons = new LinkedHashSet<>();

// 나쁜 예
LinkedHashSet<Son> sons = new LinkedHashSet<>();
```

## 2. 인터페이스를 타입으로 사용할 떄의 장점
### 1) 구현 교체 비용이 거의 0이 된다.
같은 인터페이스를 따르는 다른 구현체로 바꾸기 쉽다.
```
Set<Integer> set = new HashSet<>();     // 기본적인 고속 구현체
set = new LinkedHashSet<>();            // 순서가 필요한 상황으로 변경
set = new TreeSet<>();                  // 정렬이 필요한 상황으로 변경
```

### 2) 성능 또는 특성에 따라 적절한 구현체 선택 가능
인터페이스에 의존하면 성능/정책에 맞춘 객체 변경이 자유롭다.
- `HashMap` → `EnumMap` 사용 시 enum 기반 키에서 성능/순회 특성이 크게 향상됨
- `LinkedHashMap` 사용 시 순회 순서를 예측 가능 (LRU 캐시와 같은 패턴 구현에도 유리)

## 3. 인터페이스를 사용할 때의 주의할 점
인터페이스가 보장하지 않는 구현체 고유 기능에 의존하면, 다른 구현체로 교체할 때 문제가 발생한다.
```
Set<String> set = new LinkedHashSet<>();  // 주변 코드가 “삽입 순서 유지”를 가정하고 있다면,
set = new HashSet<>();    // 순서가 깨져 프로그램이 오작동할 수 있음
```
즉, 구현체의 특성(order, comparator 등)에 의존하는 코드는 반드시 조심해야 한다.

## 4. 인터페이스로 참조 원칙을 따르지 않아도 되는 경우
### 1) 값(value) 클래스 - String, BigInteger 
- 대체 구현이 사실상 존재하지 않고 final 이며, 인터페이스도 별도로 없다.
- 이러한 경우는 클래스를 직접 참조해도 문제가 없다.

### 2) 클래스 기반 프레임워크의 핵심 추상 클래스
- 예시 : InputStream, OutputStream, Reader, Writer, java.sql.Connection
- 이런 경우에는 인터페이스보다 추상 클래스가 역할을 정의한다. 따라서 가장 최소한의 추상 타입(상위 타입)으로 참조하면 된다.

### 3) 인터페이스에 없는 기능을 제공하는 클래스
- 예시 : PriorityQueue
- Queue 인터페이스에는 없는 comparator 접근 등 고유 기능이 있음