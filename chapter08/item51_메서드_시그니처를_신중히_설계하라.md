# item51. 메서드 시그니처를 신중히 설계하라. 


메서드 시그니처 설계 가이드를 실무 코드로 풀어보겠습니다.

## 핵심 원칙들

### 1. 메서드 이름 - 명확하고 일관되게

잘못된 예:
```java
public class UserService {
    public void proc(User u) { }  // 뭘 하는지 알 수 없음
    public void deleteUserInfo(Long id) { }  // 너무 장황함
    public void rmUsr(Long id) { }  // 축약어는 혼란만 가중
}
```

개선된 예:
```java
public class UserService {
    public void save(User user) { }
    public void delete(Long userId) { }
    public User findById(Long userId) { }
}
```

### 2. 매개변수 목록 줄이기

문제가 되는 상황을 먼저 보겠습니다:

```java
// 나쁜 예 - 매개변수가 너무 많음
public void transferMoney(String fromAccount, String toAccount, 
                          BigDecimal amount, String currency, 
                          String description, boolean urgent) {
    // 순서 헷갈리기 쉬움
}
```

**해결책 1: 메서드 분리**

List의 subList와 indexOf 조합처럼 기능을 분리합니다:

```java
// 원래 하나의 메서드로 구현한다면
public int findInRange(List<String> list, int start, int end, String target) {
    // 부분 리스트에서 특정 원소 찾기
}

// 직교성 높게 분리하면
List<String> subList = list.subList(2, 5);  // 부분 리스트 추출
int index = subList.indexOf("찾는값");      // 해당 리스트에서 찾기
```

-----

## 직교성이란?

수학에서 직교는 90도로 만나는 것, 즉 **서로 독립적**이라는 의미입니다. 소프트웨어에서는 **"기능들이 서로 영향 없이 독립적으로 조합 가능"**하다는 뜻입니다.

### 직교성이 낮은 설계 (나쁜 예)

```java
public class BadListAPI {
    // 각 조합마다 별도 메서드를 만든 경우
    public int findInRange(List<String> list, int start, int end, String target) { }
    public int findInRangeIgnoreCase(List<String> list, int start, int end, String target) { }
    public int findLastInRange(List<String> list, int start, int end, String target) { }
    public List<String> getRangeUpperCase(List<String> list, int start, int end) { }
    public List<String> getRangeLowerCase(List<String> list, int start, int end) { }
    public List<String> getRangeSorted(List<String> list, int start, int end) { }
    // ... 조합이 폭발적으로 증가!
}
```

각 기능 조합마다 메서드를 만들면 메서드 수가 기하급수적으로 늘어납니다.

### 직교성이 높은 설계 (좋은 예)

```java
// 기본 기능을 원자적으로 분리
List<String> subList = list.subList(2, 5);  // 기능 1: 부분 리스트 얻기
int index = subList.indexOf("target");       // 기능 2: 인덱스 찾기

// 이제 다양한 조합이 가능!
subList.stream().map(String::toUpperCase).collect(toList());  // 대문자 변환
Collections.sort(subList);                                     // 정렬
Collections.reverse(subList);                                  // 역순
```

**3개의 기본 기능으로 7가지 조합 가능:**
- A만, B만, C만 (3가지)
- A+B, A+C, B+C (3가지)
- A+B+C (1가지)

## 실제 예시: 도형 그리기 API

### 직교성 낮은 설계

```java
public class BadDrawingAPI {
    public void drawRedCircle(int x, int y, int radius) { }
    public void drawBlueCircle(int x, int y, int radius) { }
    public void drawRedSquare(int x, int y, int size) { }
    public void drawBlueSquare(int x, int y, int size) { }
    public void drawFilledRedCircle(int x, int y, int radius) { }
    public void drawFilledBlueCircle(int x, int y, int radius) { }
    // 색상 × 도형 × 채우기 = 조합 폭발!
}
```

### 직교성 높은 설계

```java
public class GoodDrawingAPI {
    // 기본 기능들을 독립적으로 분리
    public void setColor(Color color) { }
    public void setFilled(boolean filled) { }
    public void drawCircle(int x, int y, int radius) { }
    public void drawSquare(int x, int y, int size) { }
}

// 사용 - 원하는 대로 조합
api.setColor(Color.RED);
api.setFilled(true);
api.drawCircle(10, 10, 5);  // 빨간색 채워진 원

api.setColor(Color.BLUE);
api.setFilled(false);
api.drawSquare(20, 20, 10);  // 파란색 빈 사각형
```

## 또 다른 예시: 파일 처리 API

### 직교성 낮은 설계

```java
public class BadFileAPI {
    public String readTextFileUTF8(String path) { }
    public String readTextFileASCII(String path) { }
    public byte[] readBinaryFile(String path) { }
    public String readTextFileWithLineNumbers(String path) { }
    public String readTextFileUpperCase(String path) { }
    public void writeTextFileUTF8(String path, String content) { }
    public void writeTextFileASCII(String path, String content) { }
    // 계속 늘어남...
}
```

### 직교성 높은 설계

```java
public class GoodFileAPI {
    // 원자적 기능들
    public byte[] readBytes(String path) { }
    public void writeBytes(String path, byte[] data) { }
}

// 인코딩 처리는 별도로
public class Encoder {
    public String decode(byte[] bytes, Charset charset) { }
    public byte[] encode(String text, Charset charset) { }
}

// 텍스트 처리도 별도로
public class TextProcessor {
    public String toUpperCase(String text) { }
    public String addLineNumbers(String text) { }
}

// 사용 - 필요한 기능만 조합
byte[] bytes = fileAPI.readBytes("file.txt");
String text = encoder.decode(bytes, StandardCharsets.UTF_8);
String upper = processor.toUpperCase(text);
```

## 직교성의 장점

1. **메서드 수 감소**: n개 기능을 조합하는 것보다 n개만 만들면 됨
2. **유연성**: 예상치 못한 조합도 가능
3. **테스트 용이**: 각 기능을 독립적으로 테스트
4. **유지보수 편함**: 한 기능 수정이 다른 기능에 영향 없음

### 실무 예시: 쿼리 빌더

```java
// 직교성 높은 설계
QueryBuilder query = new QueryBuilder()
    .select("name", "age")      // 독립적 기능
    .from("users")               // 독립적 기능  
    .where("age > ?", 18)        // 독립적 기능
    .orderBy("name")             // 독립적 기능
    .limit(10);                  // 독립적 기능

// 각 메서드는 서로 독립적이라 원하는 조합 가능
QueryBuilder simpleQuery = new QueryBuilder()
    .select("*")
    .from("users");  // where, orderBy 없이도 동작
```

직교성이 높으면 적은 수의 메서드로 다양한 기능을 제공할 수 있습니다. 이게 바로 "직교성을 높여 오히려 메서드 수를 줄인다"는 의미입니다.

-----

**해결책 2: 도우미 클래스**

카드 게임 예시처럼 관련 매개변수를 묶습니다:

```java
// 나쁜 예
public void playCard(String suit, int rank) {
    // 항상 두 개를 같이 전달해야 함
}

// 개선 - 도우미 클래스 사용
public class Card {
    private final Suit suit;
    private final Rank rank;
    
    public Card(Suit suit, Rank rank) {
        this.suit = suit;
        this.rank = rank;
    }
}

public void playCard(Card card) {
    // 하나의 개념으로 묶어서 전달
}
```

송금 예시도 개선해보면:

```java
// 송금 정보를 담는 도우미 클래스
public class TransferRequest {
    private final Account from;
    private final Account to;
    private final Money amount;
    private String description;
    private boolean urgent;
}

// 깔끔해진 메서드
public void transferMoney(TransferRequest request) {
    // 구현
}
```

**해결책 3: 빌더 패턴 응용**

매개변수가 많고 일부는 선택적일 때 유용합니다:

```java
// 검색 조건 설정 예시
public class SearchCriteria {
    private String keyword;
    private LocalDate startDate;
    private LocalDate endDate;
    private int maxResults = 10;
    
    public SearchCriteria keyword(String keyword) {
        this.keyword = keyword;
        return this;
    }
    
    public SearchCriteria between(LocalDate start, LocalDate end) {
        this.startDate = start;
        this.endDate = end;
        return this;
    }
    
    public SearchCriteria maxResults(int max) {
        this.maxResults = max;
        return this;
    }
    
    public List<Result> execute() {
        // 검색 실행
        return searchService.search(this);
    }
}

// 사용 예
List<Result> results = new SearchCriteria()
    .keyword("이펙티브 자바")
    .between(LocalDate.of(2024, 1, 1), LocalDate.now())
    .maxResults(20)
    .execute();
```

### 3. 인터페이스를 매개변수로

```java
// 나쁜 예 - 특정 구현체에 의존
public void processData(HashMap<String, String> data) {
    // HashMap만 받을 수 있음
}

// 좋은 예 - 인터페이스 사용
public void processData(Map<String, String> data) {
    // HashMap, TreeMap, ConcurrentHashMap 등 모두 가능
}
```

실제 상황에서 이게 왜 중요한지 보면:

```java
// 클라이언트 코드
TreeMap<String, String> sortedData = new TreeMap<>();
// ... 데이터 추가

// 나쁜 메서드라면 복사가 필요
processData(new HashMap<>(sortedData));  // 불필요한 복사 비용!

// 좋은 메서드라면 그대로 전달
processData(sortedData);  // 바로 사용 가능
```

### 4. Boolean보다 열거 타입

```java
// 나쁜 예
public File createFile(String name, boolean temporary) {
    if (temporary) {
        // 임시 파일 생성
    } else {
        // 영구 파일 생성
    }
}

// 호출할 때
createFile("data.txt", true);  // true가 뭘 의미하는지 불명확

// 개선 - 열거 타입 사용
public enum FileType {
    TEMPORARY, 
    PERMANENT
}

public File createFile(String name, FileType type) {
    switch(type) {
        case TEMPORARY:
            // 임시 파일 생성
        case PERMANENT:
            // 영구 파일 생성
    }
}

// 호출할 때
createFile("data.txt", FileType.TEMPORARY);  // 명확함
```

온도 변환 예시도 실제로 구현하면:

```java
public enum TemperatureScale {
    FAHRENHEIT {
        public double toCelsius(double value) {
            return (value - 32) * 5 / 9;
        }
    },
    CELSIUS {
        public double toCelsius(double value) {
            return value;
        }
    },
    KELVIN {  // 나중에 추가 가능
        public double toCelsius(double value) {
            return value - 273.15;
        }
    };
    
    public abstract double toCelsius(double value);
}

// 사용
Thermometer thermometer = Thermometer.newInstance(TemperatureScale.FAHRENHEIT);
```

이런 설계 원칙들을 따르면 코드가 더 읽기 쉽고, 실수할 가능성도 줄어듭니다. 특히 협업할 때 API 문서를 계속 확인하지 않아도 직관적으로 사용할 수 있게 됩니다.

## 이해도 테스트

**문제 1.** 다음 중 직교성이 가장 높은 API 설계는?

A) `findAndSortInRange(list, start, end, target)`

B) `list.subList(start, end).sorted().find(target)`  

C) `findSortedSubList(list, start, end, target)`

D) `processListWithOptions(list, FIND|SORT|RANGE)`

----

**문제 2.** 매개변수가 6개인 메서드를 개선하는 방법이 아닌 것은?

A) 관련 매개변수를 도우미 클래스로 묶기

B) 메서드를 여러 개로 분리하기

C) 모든 매개변수를 String 타입으로 통일하기

D) 빌더 패턴을 적용하기

------

**문제 3.** `processData(HashMap<String, String> data)` 대신 `processData(Map<String, String> data)`를 사용해야 하는 이유는?

A) Map이 HashMap보다 성능이 좋아서

B) 다양한 Map 구현체를 받을 수 있어서

C) Map이 더 짧은 이름이라서

D) HashMap이 deprecated 되었기 때문에