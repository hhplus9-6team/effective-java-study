# item46. 스트림에서는 부작용 없는 함수를 사용하라

### 1. 스트림의 핵심 철학

#### 스트림 패러다임 핵심 규칙
- 계산을 연속된 변환(transform)*으로 구성한다.
- 각 변환은 순수 함수(pure function)여야 한다.
- 외부 상태 변경(부작용)은 금지한다.

#### 순수 함수란?
- 입력만 결과에 영향을 준다.
- 외부 변수/객체를 변경하지 않는다.
- 외부 가변 상태를 참조하지 않는다.

### 2. 스트림에서 부작용이 위험한 이유
forEach 등에서 외부 상태를 변경하면 스트림 철학과 어긋난다.
```
Map<String, Long> freq = new HashMap<>();

words.stream().forEach(
word -> freq.merge(word.toLowerCase(), 1L, Long::sum)
);
```
이런 코드가 문제인 이유
- 가독성 저하 : 스트림이 “변환”이 아니라 “수정”을 한다는 점이 숨겨짐
- 재사용성 낮음 : 외부 상태에 의존하면 재사용이 불가능해짐
- 테스트 어려움 : 스트림 파이프라인은 lazy + 병렬 실행되기도 함
- 동시성 문제 : 병렬 스트림에서 공유 맵 수정하여 경쟁 상태 발생

결론: forEach로 “계산”을 하는 순간 스트림의 장점이 모두 사라진다. 

### 3. 코드 진화 과정
#### 단계 1 — 사용법만 알고 억지로 스트림 사용
```
words.stream().forEach(
   word -> freq.merge(word.toLowerCase(), 1L, Long::sum)
);
```
   
#### 단계 1-1 — 스트림 제거 (오히려 for문이 더 나음)
```
for (String word : words) {
freq.merge(word.toLowerCase(), 1L, Long::sum);
}
```

#### 단계 2 — 스트림다운 방식: collect 활용
```
Map<String, Long> freq = words.stream()
.collect(groupingBy(String::toLowerCase, counting()));
```
- 문자열을 toLowerCase 로 변환하여 grouping 하고, 그룹핑된 key 에 대한 counting 이 들어갈 것이다.

### 4. Collectors API 활용
#### 4-1. toMap() (단순 맵 생성)
```
Map<String, Operation> map =
  Stream.of(Operation.values())
    .collect(toMap(e -> e.toEnglish(), e -> e));
```

#### 4-2. toMap() (병합 전략 포함)
```
Map<String, Album> topHits = albums.stream().collect(
  toMap(
    Album::artist,
    a -> a,
    BinaryOperator.maxBy(comparing(Album::sales))
  )
);
```
#### 4-3. groupingBy() (분류)
```
Map<String, List<String>> map =
words.stream().collect(groupingBy(this::alphabetize));
```
- 리스트 값을 분류하여 맵으로 만들기

#### 4-4. partitioningBy() (true/false 분류)
```
words.stream().collect(partitioningBy(w -> w.equals("stop")));
```

#### 4-5. 숫자 집계
```
students.stream().collect(summingInt(Student::score));
students.stream().collect(averagingInt(Student::score));
students.stream().collect(summarizingInt(Student::score));
```

#### 4-6. 문자열 합치기
```
food.stream().collect(joining(", ", "맛있는 ", "이 좋아"));
```

### 5. 결론
- 스트림은 순수 함수 기반의 변환 파이프라인
- forEach로 계산하지 말고 collect/reduce로 계산하라
- 부작용 없는 함수만 사용하면 병렬,재사용,테스트 모두 쉬워진다

