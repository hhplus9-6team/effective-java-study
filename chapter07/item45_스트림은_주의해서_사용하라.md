
# ITEM 45. 스트림은 주의해서 사용하라

## 스트림이란?

- 데이터 원소의 유한 또는 무한 시퀀스를 나타냄
- 스트림의 원소는 객체 참조나 기본 타입 값(int, long, double)
- 스트림은 **컬렉션과는 다르게** 데이터를 저장하지 않고, 계산에 필요한 파이프라인을 제공
- 파이프라인 하나를 구성하는 모든 호출을 연결하여 단 하나의 표현식으로 완성할 수 있다.

### 스트림 파이프라인
- **구조**: 소스 스트림 → 중간 연산 → 종단 연산
- **지연 평가(lazy evaluation)**: 종단 연산이 호출될 때까지 중간 연산이 실행되지 않음
- 무한 스트림을 다룰 수 있는 이유: 지연 평가 덕분에 필요한 만큼만 계산

**예시:**
```java
Stream.of("a", "b", "c")
    .filter(s -> s.startsWith("a"))  // 중간 연산 (지연 평가)
        .map(String::toUpperCase)        // 중간 연산 (지연 평가)
    .forEach(System.out::println);   // 종단 연산 (실제 실행)
```

**중간 연산 (intermediate operation)**
- filter(Predicate<? super T> predicate) : predicate 함수에 맞는 요소만 사용하도록 필터

- map(Function<? Super T, ? extends R> function) : 요소 각각의 function 적용

- flatMap(Function<? Super T, ? extends R> function) : 스트림의 스트림을 하나의 스트림으로 변환

- distinct() : 중복 요소 제거

- sort() : 기본 정렬

- sort(Comparator<? super T> comparator) : comparator 함수를 이용하여 정렬

- skip(long n) : n개 만큼의 스트림 요소 건너뜀

- limit(long maxSize) : maxSize 갯수만큼만 출력

**종단 연산 (terminal operation)**
- forEach(Consumer<? super T> consumer) : Stream의 요소를 순회

- count() : 스트림 내의 요소 수 반환

- max(Comparator<? super T> comparator) : 스트림 내의 최대 값 반환

- min(Comparator<? super T> comparator) : 스트림 내의 최소 값 반환

- allMatch(Predicate<? super T> predicate) : 스트림 내에 모든 요소가 predicate 함수에 만족할 경우 true

- anyMatch(Predicate<? super T> predicate) : 스트림 내에 하나의 요소라도 predicate 함수에 만족할 경우 true

- noneMatch(Predicate<? super T> predicate) : 스트림 내에 모든 요소가 predicate 함수에 만족하지않는 경우 true

- sum() : 스트림 내의 요소의 합 (IntStream, LongStream, DoubleStream)

- average() : 스트림 내의 요소의 평균 (IntStream, LongStream, DoubleStream)


## 스트림을 사용하면 안 좋은 경우

### 1. 과도한 스트림 사용 

**아나그램(Anagram)**: 철자를 재배열하여 다른 단어를 만드는 것
- 예: "staple" ↔ "petals", "stop" ↔ "pots"

```java
public class Anagrams {
   public static void main(String[] args) throws IOException {
      Path dictionary = Paths.get(args[0]);
      int minGroupSize = Integer.parseInt(args[1]);

      try (Stream<String> words = Files.lines(dictionary)) {
         words.collect(
                         groupingBy(word -> word.chars().sorted()
                                 .collect(StringBuilder::new,
                                         (sb, c) -> sb.append((char) c),
                                         StringBuilder::append).toString()))
                 .values().stream()
                 .filter(group -> group.size() >= minGroupSize)
                 .map(group -> group.size() + ": " + group)
                 .forEach(System.out::println);
      }
   }
}
```

- `word.chars().sorted().collect(...)` 부분이 너무 복잡하고 이해하기 어려움

### 2. 적절한 스트림

```java
public class Anagrams {
   public static void main(String[] args) throws IOException {
      Path dictionary = Paths.get(args[0]);
      int minGroupSize = Integer.parseInt(args[1]);

      try (Stream<String> words = Files.lines(dictionary)) {
         words.collect(groupingBy(word -> alphabetize(word)))
                 .values().stream()
                 .filter(group -> group.size() >= minGroupSize)
                 .forEach(g -> System.out.println(g.size() + ": " + g));
      }
   }

   private static String alphabetize(String s) {
      char[] a = s.toCharArray();
      Arrays.sort(a);
      return new String(a);
   }
}
```


## 스트림 사용 시 주의사항

### 1. 람다 매개변수 이름 신중하게 선택
- 타입 정보가 생략되므로 이름으로 의미를 전달해야 함
- 좋은 예: `word`, `group`, `rank`
- 나쁜 예: `w`, `g`, `r`

### 2. 도우미 메서드 적극 활용
- 복잡한 로직은 별도 메서드로 분리
- 람다보다 메서드 참조가 더 명확할 수 있음

### 3. char 값 스트림 주의
```java
"Hello world!".chars().forEach(System.out::print); // 숫자가 출력됨

// 올바른 코드
"Hello world!".chars().forEach(x -> System.out.print((char) x));
```

### 4. 기존 코드는 스트림을 사용하도록 리팩토링하되, 새 코드가 더 나아 보일 때만 반영
- 스트림과 반복문을 적절히 활용

### 스트림이 적합한 경우

1. **원소들의 시퀀스를 일관되게 변환**
2. **원소들의 시퀀스를 필터링**
3. **원소들의 시퀀스를 하나의 연산을 사용해 결합** (더하기, 연결, 최솟값 구하기 등)
4. **원소들의 시퀀스를 컬렉션에 모으기** (공통 속성 기준)
5. **원소들의 시퀀스에서 특정 조건을 만족하는 원소를 찾기**

### 반복문이 적합한 경우

1. **지역 변수를 읽고 수정해야 하는 경우**
   - 람다에서는 final이거나 사실상 final인 변수만 읽을 수 있음
   - 지역 변수를 수정할 수 없음


2. **return, break, continue가 필요한 경우**
   - 람다에서는 이런 제어 흐름을 사용할 수 없음


3. **검사 예외를 던져야 하는 경우**
   - 람다에서는 검사 예외를 직접 던질 수 없음
   - try-catch로 감싸야 함



### 예제: 데카르트 곱 계산

```java
// 반복문 버전
private static List<Card> newDeck() {
   List<Card> result = new ArrayList<>();
   for (Suit suit : Suit.values())
      for (Rank rank : Rank.values())
         result.add(new Card(suit, rank));
   return result;
}

// 스트림 버전
private static List<Card> newDeck() {
   return Stream.of(Suit.values())
           .flatMap(suit ->
                   Stream.of(Rank.values())
                           .map(rank -> new Card(suit, rank)))
           .collect(toList());
}
```

**선택 기준:** 개인 취향과 프로그래밍 환경에 따라 결정

---

## 질문

### Q1. 스트림이 적합한 경우와 반복문이 적합한 경우는?


