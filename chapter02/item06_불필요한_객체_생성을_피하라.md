# ITEM06: 불필요한 객체 생성을 막아라

## 핵심 개념
**같은 기능의 객체를 반복해서 생성하는 것은 비효율적입니다. 객체를 재사용하면 성능과 메모리 사용량을 크게 개선할 수 있습니다.**

---

## 1. 문자열 리터럴 사용하기

### ❌ 잘못된 방법
```java
String name = new String("noah");  // 매번 새로운 객체 생성
```

### ✅ 올바른 방법
```java
String name = "noah";  // 문자열 풀에서 재사용
```

### 왜 이렇게 해야 할까?

#### 문자열 풀(String Pool)의 동작 원리
```java
// 문자열 풀은 JVM의 메모리 영역 중 하나
// 같은 문자열 리터럴은 하나의 객체만 생성하고 재사용

String str1 = "hello";
String str2 = "hello";
String str3 = "hello";

// str1, str2, str3는 모두 같은 객체를 참조
System.out.println(str1 == str2);  // true - 같은 객체
System.out.println(str2 == str3);  // true - 같은 객체
```

#### new String()의 내부 동작
```java
String str = new String("hello");
// 내부적으로:
// 1. 힙 메모리에서 새로운 String 객체 생성
// 2. "hello" 문자열을 복사하여 저장
// 3. 문자열 풀과는 별개의 객체 생성
// 4. 메모리 주소 반환

// 결과: 문자열 풀의 "hello"와는 다른 객체
String poolStr = "hello";
String newStr = new String("hello");
System.out.println(poolStr == newStr);  // false - 다른 객체
```

#### 메모리 사용량 비교
```java
// 나쁜 방법: 매번 새로운 객체 생성
for (int i = 0; i < 1000000; i++) {
    String str = new String("hello");  // 100만 개의 String 객체
}
// 메모리 사용량: 100만 × String 객체 크기

// 좋은 방법: 문자열 풀에서 재사용
for (int i = 0; i < 1000000; i++) {
    String str = "hello";  // 1개의 String 객체만 사용
}
// 메모리 사용량: 1 × String 객체 크기
```



#### 가비지 컬렉션에 미치는 영향
```java
// new String() 사용 시
for (int i = 0; i < 1000000; i++) {
    String str = new String("hello");  // 100만 개 객체 생성
    // str 변수가 스코프를 벗어나면 가비지가 됨
}
// 결과: 100만 개의 가비지 객체 생성
// → 가비지 컬렉터가 정리해야 할 객체 증가
// → GC 실행 빈도 증가
// → 프로그램 성능 저하

// 문자열 리터럴 사용 시
for (int i = 0; i < 1000000; i++) {
    String str = "hello";  // 같은 객체 재사용
    // str 변수가 스코프를 벗어나도 가비지 없음
}
// 결과: 가비지 객체 생성 없음
// → 가비지 컬렉터가 할 일 없음
// → GC 실행 빈도 감소
// → 프로그램 성능 향상
```


#### JVM 메모리 구조 이해하기

**전체 메모리 구조**:
```
JVM 메모리
├── Method Area (메서드 영역)
├── Heap (힙)
│   ├── Young Generation (젊은 세대)
│   │   ├── Eden Space
│   │   ├── Survivor Space 0
│   │   └── Survivor Space 1
│   └── Old Generation (늙은 세대)
├── Stack (스택)
├── PC Register (PC 레지스터)
└── Native Method Stack (네이티브 메서드 스택)
```

**힙(Heap) 메모리**:
- 모든 객체가 저장되는 메모리 영역
- Young Generation과 Old Generation으로 구분
- 가비지 컬렉션의 대상

**문자열 풀(String Pool)**:
- 힙 내부의 특별한 영역
- 문자열 리터럴을 저장하고 재사용
- 메모리 절약과 성능 향상

#### 메모리 위치의 차이
```java
String poolStr = "hello";        // 문자열 풀에 저장
String newStr = new String("hello");  // 힙 메모리에 새로 생성

```




---

## 2. 정적 팩터리 메서드 사용하기

### ❌ 잘못된 방법
```java
Boolean bool = new Boolean(true);  // 매번 새로운 객체 생성
Integer num = new Integer(42);     
```

### ✅ 올바른 방법
```java
Boolean bool = Boolean.valueOf(true);  // 캐시된 객체 재사용
Integer num = Integer.valueOf(42);     
```

### 왜 이렇게 해야 할까?

#### Boolean.valueOf()의 내부 동작
```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}

// Boolean 클래스 내부에 미리 정의된 상수들
public static final Boolean TRUE = new Boolean(true);
public static final Boolean FALSE = new Boolean(false);
```

**핵심**: `Boolean.valueOf()`는 내부적으로 미리 생성된 `Boolean.TRUE`와 `Boolean.FALSE` 상수를 반환합니다. 따라서 매번 새로운 객체를 생성하지 않습니다.

#### Integer.valueOf()의 캐싱 메커니즘
```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}

// IntegerCache 내부 구조
private static class IntegerCache {
    static final int low = -128;
    static final int high = 127;
    static final Integer cache[];
    
    static {
        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);
    }
}
```

**핵심**:
- `-128`부터 `127`까지의 정수는 미리 생성되어 캐시에 저장됩니다
- 이 범위를 벗어나면 새로운 `Integer` 객체를 생성합니다
- 이는 자주 사용되는 작은 정수들의 메모리 사용량을 줄이기 위한 최적화입니다



#### 메모리 사용량 비교
```java
// 나쁜 방법: 매번 새로운 객체 생성
for (int i = 0; i < 1000000; i++) {
    Boolean flag = new Boolean(true);  // 100만 개의 Boolean 객체
}
// 메모리 사용량: 100만 × Boolean 객체 크기

// 좋은 방법: 캐시된 객체 재사용
for (int i = 0; i < 1000000; i++) {
    Boolean flag = Boolean.valueOf(true);  // 1개의 Boolean 객체만 사용
}
// 메모리 사용량: 1 × Boolean 객체 크기
```

#### JVM 레벨에서의 차이점

**객체 생성 과정**:
1. **메모리 할당**: 힙 메모리에서 객체 크기만큼 공간 할당
2. **객체 초기화**: 생성자 호출하여 필드 초기화
3. **메모리 주소 반환**: 생성된 객체의 참조 반환

**캐시된 객체 재사용**:
1. **캐시 조회**: 미리 생성된 객체의 참조 반환
2. **메모리 할당 없음**: 새로운 메모리 할당 불필요
3. **초기화 없음**: 이미 초기화된 객체 사용


---

## 3. 비싼 객체 재사용하기

### 정규표현식 예시


#### ❌ 잘못된 방법
```java
public class PhoneNumber {
    // 나쁜 예: 매번 정규표현식 컴파일
    static boolean isValid(String s) {
        return s.matches("^[0-9]{3}-[0-9]{4}-[0-9]{4}$");
    }
}
```

#### ✅ 올바른 방법
```java
public class PhoneNumber {
    // 좋은 예: 정규표현식 패턴 재사용
    private static final Pattern PHONE_PATTERN = 
        Pattern.compile("^[0-9]{3}-[0-9]{4}-[0-9]{4}$");
    
    static boolean isValidGood(String s) {
        return PHONE_PATTERN.matcher(s).matches();
    }
}
```

#### 왜 이렇게 해야 할까?

**정규표현식 컴파일 과정**:
```java
Pattern.compile("^[0-9]{3}-[0-9]{4}-[0-9]{4}$")
// 내부적으로 일어나는 일:

// 1단계: 정규표현식 파싱
// "010-1234-5678" 패턴을 분석

// 2단계: NFA(Non-deterministic Finite Automaton) 생성
// 비결정적 유한상태자동기 생성

// 3단계: DFA(Deterministic Finite Automaton) 변환  
// 결정적 유한상태자동기로 변환

// 4단계: 상태 전환 테이블 생성
// 각 상태에서 어떤 입력에 어떤 상태로 갈지 테이블 생성

// 5단계: 최적화
// 불필요한 상태 제거, 성능 최적화

// → 이 모든 과정이 매우 비싸다! (느리다!)
```


**실제 성능 측정**:
```java
public class RegexPerformanceTest {
    public static void main(String[] args) {
        List<String> phones = generatePhones(100000); // 10만 개 전화번호
        
        // 나쁜 방법: 매번 컴파일
        long start1 = System.currentTimeMillis();
        for (String phone : phones) {
            phone.matches("^[0-9]{3}-[0-9]{4}-[0-9]{4}$");
        }
        long end1 = System.currentTimeMillis();
        System.out.println("매번 컴파일: " + (end1 - start1) + "ms");
        
        // 좋은 방법: 미리 컴파일
        Pattern pattern = Pattern.compile("^[0-9]{3}-[0-9]{4}-[0-9]{4}$");
        long start2 = System.currentTimeMillis();
        for (String phone : phones) {
            pattern.matcher(phone).matches();
        }
        long end2 = System.currentTimeMillis();
        System.out.println("미리 컴파일: " + (end2 - start2) + "ms");
    }
}

// 결과 예시:
// 매번 컴파일: 5000ms
// 미리 컴파일: 50ms
#### 지연 초기화에 대한 고려사항

**지연 초기화란?**
객체가 실제로 사용될 때까지 초기화를 미루는 기법

```java
// 즉시 초기화 (현재 방식)
public class RomanNumeralValidator {
    private static final Pattern ROMAN = Pattern.compile("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
    
    static boolean isRomanNumeral(String s) {
        return ROMAN.matcher(s).matches();
    }
}

// 지연 초기화 방식
public class RomanNumeralValidatorLazy {
    private static Pattern ROMAN;  // 초기화하지 않음
    
    static boolean isRomanNumeral(String s) {
        if (ROMAN == null) {  // 처음 호출될 때만 초기화
            ROMAN = Pattern.compile("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
        }
        return ROMAN.matcher(s).matches();
    }
}
```

**지연 초기화를 권하지 않는 이유**:
1. **코드 복잡성**: null 체크, 동기화 등으로 복잡해짐
2. **스레드 안전성**: 동시성 문제 발생 가능
3. **성능 오버헤드**: 매번 체크하는 비용
4. **유지보수성**: 버그 발생 가능성 증가

**결론**: 대부분의 경우 즉시 초기화가 더 좋다!

#### Map의 어댑터 패턴 예시

**문제가 되는 코드**:
```java
public class MapExample {
    private final Map<String, Object> map = new HashMap<>();
    
    // 나쁜 방법: 매번 새로운 Set 생성
    public Set<String> getKeys() {
        return new HashSet<>(map.keySet());  // 매번 새로운 Set 객체 생성
    }
    
    public Collection<Object> getValues() {
        return new ArrayList<>(map.values());  // 매번 새로운 Collection 생성
    }
}
```

**개선된 코드**:
```java
public class ImprovedMapExample {
    private final Map<String, Object> map = new HashMap<>();
    
    // 어댑터들을 미리 생성해서 재사용
    private Set<String> keySet;
    private Collection<Object> valueCollection;
    
    public Set<String> getKeys() {
        if (keySet == null) {
            keySet = new HashSet<>(map.keySet());
        }
        return keySet;  // 같은 Set 재사용
    }
    
    public Collection<Object> getValues() {
        if (valueCollection == null) {
            valueCollection = new ArrayList<>(map.values());
        }
        return valueCollection;  // 같은 Collection 재사용
    }
}
```

---

## 4. 오토박싱 주의하기

### ❌ 잘못된 방법
```java
Long sum = 0L;  // 박싱된 타입 사용
for (long i = 0; i < Integer.MAX_VALUE; i++) {
    sum += i;  // 매번 Long 객체 생성 (오토박싱)
}
```

### ✅ 올바른 방법
```java
long sum = 0L;  // 기본 타입 사용
for (long i = 0; i < Integer.MAX_VALUE; i++) {
    sum += i;  // 오토박싱 없음
}
```

### 왜 이렇게 해야 할까?

#### 오토박싱이란?
**기본 타입을 래퍼 클래스로 자동 변환하는 것**

```java
// 오토박싱 예시
int primitive = 42;           // 기본 타입
Integer boxed = primitive;    // 자동으로 박싱됨

// 오토언박싱 예시
Integer boxed = 42;           // 박싱된 타입
int primitive = boxed;        // 자동으로 언박싱됨
```

#### 박싱된 타입 vs 기본 타입

**박싱된 타입 (래퍼 클래스)**:
```java
Integer number = 42;        // 박싱된 타입
Long bigNumber = 123L;      // 박싱된 타입
Boolean flag = true;        // 박싱된 타입
Double price = 99.99;       // 박싱된 타입
```

**기본 타입**:
```java
int number = 42;            // 기본 타입
long bigNumber = 123L;      // 기본 타입
boolean flag = true;        // 기본 타입
double price = 99.99;       // 기본 타입
```

#### 오토박싱의 성능 문제

**문제가 되는 코드**:
```java
Long sum = 0L;  // 박싱된 타입
for (long i = 0; i < 1000000; i++) {
    sum += i;  // 매번 오토박싱 발생!
    // 내부적으로: sum = Long.valueOf(sum.longValue() + i);
}
```

**해결된 코드**:
```java
long sum = 0L;  // 기본 타입
for (long i = 0; i < 1000000; i++) {
    sum += i;  // 오토박싱 없음!
}
```

#### 실제 성능 차이
```java
public class BoxingPerformanceTest {
    public static void main(String[] args) {
        int iterations = 10000000;
        
        // 박싱된 타입 사용
        Long sum1 = 0L;
        long start1 = System.nanoTime();
        for (long i = 0; i < iterations; i++) {
            sum1 += i;  // 매번 오토박싱
        }
        long end1 = System.nanoTime();
        System.out.println("박싱된 타입: " + (end1 - start1) / 1000000 + "ms");
        
        // 기본 타입 사용
        long sum2 = 0L;
        long start2 = System.nanoTime();
        for (long i = 0; i < iterations; i++) {
            sum2 += i;  // 오토박싱 없음
        }
        long end2 = System.nanoTime();
        System.out.println("기본 타입: " + (end2 - start2) / 1000000 + "ms");
    }
}

// 결과 예시:
// 박싱된 타입: 2000ms
// 기본 타입: 50ms
// → 40배 성능 차이!
```

#### 언제 박싱된 타입을 써야 할까?

**✅ 박싱된 타입이 필요한 경우**:
```java
// 1. 컬렉션에 저장할 때
List<Integer> numbers = new ArrayList<>();
numbers.add(42);  // int는 컬렉션에 저장 불가

// 2. null 값이 필요할 때
Integer score = null;  // 기본 타입은 null 불가
if (score != null) {
    System.out.println("점수: " + score);
}

// 3. 제네릭 사용할 때
Map<String, Integer> scores = new HashMap<>();
```


---
