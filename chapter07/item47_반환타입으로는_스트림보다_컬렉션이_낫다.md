# item47. 반환타입으로는 스트림보다 컬렉션이 낫다.

이 아이템은 "메서드가 여러 개의 값을 반환할 때 어떤 타입을 써야 하나"에 대한 이야기다.

## 문제 상황

자바 7까지는 간단했어. 여러 값을 반환할 때:
- 기본적으로 `List<String>` 같은 컬렉션 반환
- for-each문만 쓸 거면 `Iterable<String>` 반환
- 배열도 가능

그런데 자바 8에서 스트림이 나오면서 복잡해졌어.

## 왜 복잡해졌나?

Stream은 for-each문으로 직접 못 돌려. 이게 문제야.

```java
// 이렇게 하고 싶지만 안 됨
Stream<String> names = getNames();
for (String name : names) {  // 컴파일 에러!
    System.out.println(name);
}
```

왜 안 될까요? Stream이 Iterable 인터페이스를 구현(확장)하지 않아서입니다. 신기한 건 Stream이 Iterable이 요구하는 메서드들을 다 가지고 있는데도 불구하고 말입니다.

## 해결 방법들

**1. 어댑터 패턴 - Stream을 Iterable로**

```java
// 어댑터 메서드 만들기
public static <E> Iterable<E> iterableOf(Stream<E> stream) {
    return stream::iterator;
}

// 사용
for (String name : iterableOf(getNames())) {
    System.out.println(name);
}
```

**2. 반대 방향 - Iterable을 Stream으로**

```java
public static <E> Stream<E> streamOf(Iterable<E> iterable) {
    return StreamSupport.stream(iterable.spliterator(), false);
}

// 사용
streamOf(someIterable)
    .filter(x -> x.length() > 5)
    .collect(Collectors.toList());
```

## 그래서 뭘 써야 하나?

**답: Collection을 반환해라**

왜냐하면 Collection은:
- Iterable을 상속받음 → for-each 가능
- stream() 메서드 제공 → 스트림 파이프라인 가능

```java
// 이게 제일 좋음
public Collection<String> getNames() {
    return List.of("김철수", "이영희", "박민수");
}

// 사용하는 쪽에서 자유롭게
Collection<String> names = getNames();

// 1. 반복문 쓰고 싶으면
for (String name : names) {
    System.out.println(name);
}

// 2. 스트림 쓰고 싶으면
names.stream()
    .filter(n -> n.startsWith("김"))
    .forEach(System.out::println);
```

## 실전 예시: 멱집합 구현

집합 {a, b, c}의 멱집합은 모든 부분집합을 모은 것입니다.:
`{{}, {a}, {b}, {c}, {a,b}, {a,c}, {b,c}, {a,b,c}}`

원소가 n개면 멱집합은 2^n입니다.. 만약 원소가 30개면? 2^30 = 약 10억 개. 메모리에 다 올리면 터집니다.

그래서 "게으른 컬렉션"을 만듭니다.:

```java
public static <E> Collection<Set<E>> powerSet(Set<E> s) {
    List<E> src = new ArrayList<>(s);
    if (src.size() > 30)
        throw new IllegalArgumentException("원소가 너무 많아요");

    return new AbstractList<Set<E>>() {
        @Override
        public int size() {
            return 1 << src.size();  // 2^n
        }

        @Override
        public Set<E> get(int index) {
            Set<E> result = new HashSet<>();
            // index를 비트로 해석
            // 예: index=5 (이진수 101) → 0번째, 2번째 원소 포함
            for (int i = 0; index != 0; i++, index >>= 1) {
                if ((index & 1) == 1)
                    result.add(src.get(i));
            }
            return result;
        }
    };
}
```

이렇게 하면:
- 실제 부분집합들을 미리 만들지 않음
- get(index)가 호출될 때만 그때그때 생성
- 메모리 절약하면서도 Collection 인터페이스 제공

```java
// 사용 예
Set<String> s = Set.of("a", "b", "c");
Collection<Set<String>> power = powerSet(s);

// for-each로 순회 가능
for (Set<String> subset : power) {
    System.out.println(subset);
}

// 스트림으로도 가능
power.stream()
    .filter(subset -> subset.size() == 2)
    .forEach(System.out::println);
```

-----
## 해결책 심화 버전
### 1단계: 기본 해결책 - Collection 반환

```java
// 이게 기본
public Collection<String> getNames() {
    return List.of("김철수", "이영희", "박민수");
}
```

Collection이면 for-each도 되고 stream도 되니까 만능입니다.

### 2단계: 문제 발생 - 데이터가 너무 클 때

```java
// 만약 이렇게 하면?
public Collection<Set<String>> powerSet(Set<String> s) {
    // 모든 부분집합을 실제로 만들어서 리스트에 담기
    List<Set<String>> result = new ArrayList<>();
    // ... 2^n개의 부분집합을 전부 생성
    return result;
}

// 원소 30개면? 2^30 = 10억 개
// 메모리 터짐 💥
```

### 3단계: 최종 해결책 - 전용 컬렉션 구현

"필요할 때만 만들자!" (lazy evaluation)

```java
public static <E> Collection<Set<E>> powerSet(Set<E> s) {
    List<E> src = new ArrayList<>(s);

    // AbstractList를 상속받아 전용 컬렉션 만들기
    return new AbstractList<Set<E>>() {

        // 크기만 계산 (실제로 만들지 않음)
        @Override
        public int size() {
            return 1 << src.size();  // 2^n
        }

        // 필요할 때만 i번째 부분집합 생성
        @Override
        public Set<E> get(int index) {
            Set<E> result = new HashSet<>();
            for (int i = 0; index != 0; i++, index >>= 1) {
                if ((index & 1) == 1)
                    result.add(src.get(i));
            }
            return result;
        }
    };
}
```

## 왜 전용 컬렉션이 필요한가?

**메모리 비교:**

```java
// 나쁜 방법: 전부 미리 생성
Set<String> s = Set.of("a", "b", "c", ..., "z");  // 26개
Collection<Set<String>> bad = createAllSubsets(s);
// 메모리: 2^26개 = 67,108,864개를 전부 저장 💀

// 좋은 방법: 필요할 때만 생성
Collection<Set<String>> good = powerSet(s);
// 메모리: src 배열 26개만 저장 ✅
// get(100)을 호출하면? → 그때 100번째 부분집합만 계산해서 반환
```

## 동작 예시

```java
Set<String> s = Set.of("a", "b", "c");
Collection<Set<String>> power = powerSet(s);

// 이 시점엔 아무것도 안 만들어짐
System.out.println(power.size());  // 8 출력 (계산만 함)

// 이제 실제로 필요한 것만 생성
Set<String> first = power.get(0);   // {} 생성
Set<String> fifth = power.get(5);   // {a, c} 생성

// for-each 돌릴 때도
for (Set<String> subset : power) {
    // 각 반복마다 하나씩 생성됨
    System.out.println(subset);
}

// stream 쓸 때도
power.stream()
    .filter(subset -> subset.size() == 2)  // 필요한 것만 생성
    .forEach(System.out::println);
```

## 핵심 정리

**Item 47의 3단계 해법:**

1. **작은 데이터**: `List.of()`, `Set.of()` 같은 표준 컬렉션 반환
   ```java
   return List.of(1, 2, 3, 4, 5);
   ```

2. **큰 데이터 (메모리에 올릴 수 있음)**: ArrayList, HashSet 반환
   ```java
   List<String> result = new ArrayList<>();
   // 데이터 추가...
   return result;
   ```

3. **매우 큰 데이터 (메모리 부족 우려)**: **전용 컬렉션 구현**
   ```java
   return new AbstractList<T>() {
       // size(), get()만 구현
       // 실제 데이터는 필요할 때 생성
   };
   ```

전용 컬렉션의 장점:
- Collection 인터페이스 제공 → for-each, stream 둘 다 가능
- Lazy evaluation → 메모리 절약
- 필요한 메서드만 구현하면 됨 (AbstractList 덕분)

------
## 최종 핵심 정리

1. 내부에서만 쓸 거면: Stream이나 Iterable 중 편한 거 써
2. 공개 API라면: **무조건 Collection 반환**
3. 데이터가 너무 크면: 전용 컬렉션 구현해서 게으르게 처리

Collection을 쓰면 사용자가 알아서 원하는 방식(for-each든 stream이든)으로 쓸 수 있습니다.

-----

## 문제 1 
다음 중 Stream을 for-each 문으로 직접 반복할 수 없는 이유는?

① Stream이 Iterable의 메서드를 하나도 가지고 있지 않아서
② Stream이 Iterable 인터페이스를 확장(extends)하지 않아서
③ Stream은 일회용이라 반복이 불가능해서
④ for-each 문이 Stream을 지원하지 않도록 설계되어서
⑤ Stream은 병렬 처리만 가능해서

---

## 문제 2 
공개 API에서 원소 시퀀스를 반환할 때 권장되는 반환 타입은?

① Stream<E>
② Iterable<E>
③ Collection<E>
④ List<E>
⑤ ArrayList<E>

## 문제 3 
다음 코드의 출력 결과는?

```java
Set<String> s = Set.of("a", "b");
Collection<Set<String>> power = PowerSet.of(s);

System.out.println(power.size());
```

① 2
② 3
③ 4
④ 컴파일 에러
⑤ 런타임 에러

---

## 문제 4 
다음 중 전용 컬렉션 구현이 필요한 경우는?

① 반환할 원소가 5개 이하일 때
② 반환할 시퀀스가 크지만 표현을 간결하게 할 수 있을 때
③ Stream만 사용할 것이 확실할 때
④ 성능이 중요하지 않을 때
⑤ 모든 경우에 항상 전용 컬렉션을 구현해야 함
