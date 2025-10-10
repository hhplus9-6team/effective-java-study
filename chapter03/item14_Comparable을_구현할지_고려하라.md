# ITEM14: Comparable을 구현할지 고려하라

## 1. Comparable 인터페이스

### Comparable이란?

- Comparable은 객체의 순서를 정하기 위한 Java 인터페이스입니다.

```java
public interface Comparable<T> {
    int compareTo(T o);
}
```

### **compareTo 특징**
- 단순 동치성 비교 + 순서 비교 + 제네릭 지원

---

## 2. compareTo 메서드의 일반 규약

### **반환값의 의미**
- 이 객체가 주어진 객체보다 **작으면 음수** 반환
- 이 객체가 주어진 객체와 **같으면 0** 반환
- 이 객체가 주어진 객체보다 **크면 양수** 반환
- 비교할 수 없는 타입의 객체가 주어지면 **ClassCastException** 던짐

### **규약 세 가지**

1. **대칭성**: `sgn(x.compareTo(y)) == -sgn(y.compareTo(x))`
   - x.compareTo(y)는 y.compareTo(x)가 예외를 던질 때만 예외를 던져야 함

2. **추이성**: `(x.compareTo(y) > 0 && y.compareTo(z) > 0)`이면 `x.compareTo(z) > 0`

3. **일관성**: `x.compareTo(y) == 0`이면 `sgn(x.compareTo(z)) == sgn(y.compareTo(z))`

### **권고사항 (필수는 아님)**
- `(x.compareTo(y) == 0) == (x.equals(y))`
- compareTo로 수행한 동치성 테스트 결과가 equals와 같아야 함
- 일치하지 않아도 동작하지만, 정렬된 컬렉션에 넣으면 해당 컬렉션이 구현한 인터페이스에 정의된 동작과 달라질 수 있음

---

## 3. compareTo와 equals의 일관성

### **BigDecimal 예시**
```java
BigDecimal bd1 = new BigDecimal("1.0");
BigDecimal bd2 = new BigDecimal("1.00");

// equals 비교
bd1.equals(bd2);  // false (정밀도가 다름)

// compareTo 비교
bd1.compareTo(bd2);  // 0 (값이 같음)
```

### **1. equals - "완전히 똑같은가?"**

**왜 false일까?**
- `equals`는 **값 + 정밀도(scale)** 모두를 비교
- `"1.0"` → 정밀도 1 (소수점 첫째 자리)
- `"1.00"` → 정밀도 2 (소수점 둘째 자리)
- 값은 같지만 **정밀도가 다르므로** `false`

### **2. compareTo - "수학적으로 같은가?"**

**왜 0일까?**
- `compareTo`는 **수학적인 값만** 비교
- `1.0`과 `1.00`은 수학적으로 같은 값
- 정밀도는 무시하고 **값만 같으므로** `0` (같음)

### **HashSet vs TreeSet에서의 차이**

```java
// HashSet 사용 (equals 기반)
Set<BigDecimal> hashSet = new HashSet<>();
hashSet.add(bd1);
hashSet.add(bd2);
System.out.println(hashSet.size());  // 2 (둘 다 저장)

// TreeSet 사용 (compareTo 기반)
Set<BigDecimal> treeSet = new TreeSet<>();
treeSet.add(bd1);
treeSet.add(bd2);
System.out.println(treeSet.size());  // 1 (하나만 저장)
```

**왜 이런 차이가 발생할까?**

**HashSet - equals 사용**
- HashSet은 `equals()`로 중복 판단
- `"1.0".equals("1.00")` → `false`
- 다른 객체로 인식 → **둘 다 저장**

**TreeSet - compareTo 사용**
- TreeSet은 `compareTo()`로 중복 판단
- `"1.0".compareTo("1.00")` → `0` (같음)
- 같은 객체로 인식 → **하나만 저장**

---

## 4. compareTo 메서드 작성 요령

### **1. 객체 참조 필드가 하나뿐인 경우**
```java
public class CaseInsensitiveString implements Comparable<CaseInsensitiveString> {
    private String s;
    
    @Override
    public int compareTo(CaseInsensitiveString cis) {
        return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
    }
}
```

### **2. 핵심 필드가 여러 개인 경우 - 핵심 필드부터 비교**
```java
public class PhoneNumber implements Comparable<PhoneNumber> {
    private final short areaCode;
    private final short prefix;
    private final short lineNum;
    
    @Override
    public int compareTo(PhoneNumber pn) {
        int result = Short.compare(areaCode, pn.areaCode);  // 가장 중요한 필드
        if (result == 0) {
            result = Short.compare(prefix, pn.prefix);  // 두 번째로 중요한 필드
            if (result == 0) {
                result = Short.compare(lineNum, pn.lineNum);  // 세 번째로 중요한 필드
            }
        }
        return result;
    }
}
```

### **3. 비교자 생성 메서드 활용 (Java 8)**
```java
public class PhoneNumber implements Comparable<PhoneNumber> {
    private final short areaCode;
    private final short prefix;
    private final short lineNum;
    
    private static final Comparator<PhoneNumber> COMPARATOR =
        Comparator.comparingInt((PhoneNumber pn) -> pn.areaCode)
                  .thenComparingInt(pn -> pn.prefix)
                  .thenComparingInt(pn -> pn.lineNum);
    
    @Override
    public int compareTo(PhoneNumber pn) {
        return COMPARATOR.compare(this, pn);
    }
}
```

### **4. 주의사항 - 뺄셈 금지**
```java
// 잘못된 방법 - 정수 오버플로 위험
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return o1.hashCode() - o2.hashCode();  
    }
};

// 올바른 방법 1 - 정적 compare 메서드
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return Integer.compare(o1.hashCode(), o2.hashCode());
    }
};

// 올바른 방법 2 - 비교자 생성 메서드
static Comparator<Object> hashCodeOrder = 
    Comparator.comparingInt(o -> o.hashCode());
```

---

## 5. Comparable의 활용

```java
String[] words = {"banana", "apple", "cherry"};
Arrays.sort(words);  //compareTo()메서드 자동호출
System.out.println(Arrays.toString(words));  // [apple, banana, cherry]


public class Person implements Comparable<Person> {
    private String name;
    private int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    @Override
    public int compareTo(Person other) {
        return Integer.compare(this.age, other.age);
    }
}

List<Person> people = Arrays.asList(
    new Person("홍길동", 30),
    new Person("김철수", 20),
    new Person("이영희", 25)
);
Collections.sort(people);  // age 기준으로 자동 정렬
```

---
## 질문

### Q1. compareTo와 equals의 일관성이 지켜지지 않으면 어떤 문제가 발생하나요?
### Q2. compareTo 메서드 구현 시 값의 차이를 뺄셈으로 반환하면 안 되는 이유는 무엇인가요?