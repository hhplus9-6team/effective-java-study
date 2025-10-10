# item13. clone 재정의는 주의해서 진행해라

# Cloneable과 clone() 가이드

## 📌 1. Cloneable의 구조적 문제

### 문제 1: 인터페이스인데 메서드가 없음
```java
public interface Cloneable {
    // 텅 비어있음!
}
```
- 보통 인터페이스는 구현할 메서드를 정의하는데, `Cloneable`은 아무것도 없음
- 실제 `clone()` 메서드는 `Object` 클래스에 있음

### 문제 2: clone() 메서드의 이상한 위치
```java
public class Object {
    protected Object clone() throws CloneNotSupportedException {
        // ...
    }
}
```
- `clone()`이 `protected`라서 외부에서 바로 호출 불가
- `Cloneable` 구현 여부에 따라 동작이 달라짐
    - 구현O → 필드 복사
    - 구현X → `CloneNotSupportedException` 발생

### 문제 3: 이게 뭔 인터페이스인가?
```java
class Phone implements Cloneable {
    // Cloneable을 구현해도 clone() 메서드는 여전히 protected
    // public으로 직접 재정의해야 외부에서 사용 가능
}
```
**일반적인 인터페이스:** "이 기능 구현해주세요"  
**Cloneable:** "Object의 protected 메서드 동작 방식만 바꿔줄게요" ← 매우 이례적!

---

## 📌 2. clone() 메서드 규약의 문제점

### 규약 내용 (느슨함 주의!)
```java
// 1. 복제본은 원본과 다른 객체여야 함
x.clone() != x  // 참이어야 함

// 2. 같은 클래스여야 함
x.clone().getClass() == x.getClass()  // 참이어야 함

// 3. equals로 비교하면 같아야 함 (하지만 필수 아님!)
x.clone().equals(x)  // 참이면 좋음
```

### 핵심 규약: super.clone() 호출
```java
class Parent implements Cloneable {
    @Override
    public Parent clone() {
        return new Parent();  // ❌ 생성자 사용 - 규약 위반
    }
}

class Child extends Parent {
    @Override
    public Child clone() {
        return (Child) super.clone();  // ❌ Parent 객체 반환됨!
    }
}
```

**올바른 방법:**
```java
class Parent implements Cloneable {
    @Override
    public Parent clone() {
        return (Parent) super.clone();  // ✅ super.clone() 사용
    }
}

class Child extends Parent {
    @Override
    public Child clone() {
        return (Child) super.clone();  // ✅ 제대로 작동
    }
}
```

**예외:** `final` 클래스는 하위 클래스가 없으니 생성자 써도 OK

---

## 📌 3. 가변 상태 복제 - 단계별 난이도

### Level 1: 기본 타입 / 불변 객체만 있는 경우 (쉬움)

```java
public class PhoneNumber implements Cloneable {
    private final int areaCode;     // 기본 타입
    private final int prefix;
    private final int lineNum;
    
    @Override 
    public PhoneNumber clone() {
        try {
            // super.clone()만으로 충분!
            return (PhoneNumber) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();  // Cloneable 구현했으니 발생 안 함
        }
    }
}
```
**포인트:**
- 모든 필드가 기본 타입이거나 불변 객체면 끝!
- `super.clone()`이 모든 필드를 자동으로 복사해줌
- 공변 반환 타이핑으로 `PhoneNumber` 타입 반환 (형변환 불필요)

---

### Level 2: 가변 객체(배열) 참조하는 경우 (중간)

```java
public class Stack implements Cloneable {
    private Object[] elements;
    private int size = 0;
    
    public void push(Object e) { /* ... */ }
    public Object pop() { /* ... */ }
    
    // ❌ 잘못된 방법
    @Override
    public Stack clone() {
        Stack result = (Stack) super.clone();
        // elements는 원본과 같은 배열을 참조!
        // 복제본 수정 시 원본도 영향받음
        return result;
    }
    
    // ✅ 올바른 방법
    @Override
    public Stack clone() {
        try {
            Stack result = (Stack) super.clone();
            result.elements = elements.clone();  // 배열도 복제!
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

**문제 상황:**
```java
Stack original = new Stack();
original.push("A");
original.push("B");

Stack copy = original.clone();  // 잘못된 clone 사용 시
copy.pop();  // "B" 제거

// ❌ 원본도 영향받음!
System.out.println(original.size());  // 1로 변경됨
```

**배열 복제의 특징:**
- 배열의 `clone()`은 런타임/컴파일타임 타입 모두 원본과 동일하게 반환
- 형변환 불필요
- 배열은 `clone()` 사용이 가장 깔끔한 유일한 예외

---

### Level 3: 복잡한 가변 객체(연결 리스트 등) (어려움)

```java
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;
    
    private static class Entry {
        final Object key;
        Object value;
        Entry next;  // 연결 리스트!
        
        Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }
        
        // ❌ 재귀 방식 - 리스트가 길면 스택 오버플로우
        Entry deepCopy() {
            return new Entry(key, value, 
                next == null ? null : next.deepCopy());
        }
        
        // ✅ 반복문 방식 - 안전함
        Entry deepCopy() {
            Entry result = new Entry(key, value, next);
            for (Entry p = result; p.next != null; p = p.next) {
                p.next = new Entry(p.next.key, p.next.value, p.next.next);
            }
            return result;
        }
    }
    
    // ❌ 잘못된 방법
    @Override 
    public HashTable clone() {
        HashTable result = (HashTable) super.clone();
        result.buckets = buckets.clone();  // 배열만 복제
        // 각 Entry의 연결 리스트는 원본과 공유됨!
        return result;
    }
    
    // ✅ 올바른 방법
    @Override 
    public HashTable clone() {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];
            
            // 각 버킷의 연결 리스트를 깊은 복사
            for (int i = 0; i < buckets.length; i++) {
                if (buckets[i] != null) {
                    result.buckets[i] = buckets[i].deepCopy();
                }
            }
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

**깊은 복사가 필요한 이유:**
```java
HashTable original = new HashTable();
original.put("key1", "value1");
original.put("key2", "value2");

HashTable copy = original.clone();  // 얕은 복사 시
copy.put("key1", "MODIFIED");

// ❌ 원본도 변경됨!
System.out.println(original.get("key1"));  // "MODIFIED"
```

---

### Level 4: 고수준 메서드 활용 (대안)

```java
@Override 
public HashTable clone() {
    try {
        HashTable result = (HashTable) super.clone();
        result.buckets = new Entry[buckets.length];  // 초기화
        
        // 고수준 메서드로 복사
        for (Entry entry : buckets) {
            for (Entry e = entry; e != null; e = e.next) {
                result.put(e.key, e.value);  // put 메서드 활용
            }
        }
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

**장점:** 코드가 간단하고 우아함  
**단점:**
- 저수준 처리보다 느림
- `put()` 같은 메서드가 `final`이어야 함 (하위 클래스에서 재정의 방지)

---

## 📌 4. final 필드와의 충돌

```java
public class Stack implements Cloneable {
    private final Object[] elements;  // final!
    private int size;
    
    @Override 
    public Stack clone() {
        Stack result = (Stack) super.clone();
        result.elements = elements.clone();  // ❌ 컴파일 에러!
        // final 필드는 재할당 불가
        return result;
    }
}
```

**해결 방법:**
```java
public class Stack implements Cloneable {
    private Object[] elements;  // final 제거
    private int size;
    
    @Override 
    public Stack clone() {
        Stack result = (Stack) super.clone();
        result.elements = elements.clone();  // ✅ 작동
        return result;
    }
}
```

**문제:** "가변 객체를 참조하는 필드는 `final`로 선언하라"는 일반 원칙과 충돌!

---

## 📌 5. 주의사항 체크리스트

### ⚠️ 1. 재정의 가능한 메서드 호출 금지
```java
class Parent implements Cloneable {
    @Override
    public Parent clone() {
        Parent result = (Parent) super.clone();
        result.init();  // ❌ 하위 클래스에서 재정의 가능
        return result;
    }
    
    protected void init() { /* ... */ }
}

class Child extends Parent {
    private List<String> data;
    
    @Override
    protected void init() {
        this.data = new ArrayList<>();  // 복제 과정에서 호출되면 문제!
    }
}
```

**해결:** `init()` 메서드를 `final` 또는 `private`으로 만들기

---

### ⚠️ 2. 검사 예외 제거
```java
// ❌ 불편함
@Override 
public PhoneNumber clone() throws CloneNotSupportedException {
    return (PhoneNumber) super.clone();
}

// ✅ 편리함
@Override 
public PhoneNumber clone() {  // throws 절 제거
    try {
        return (PhoneNumber) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();  // 발생할 수 없음
    }
}
```

---

### ⚠️ 3. 스레드 안전성
```java
public class SynchronizedList implements Cloneable {
    private List<String> data;
    
    // ❌ 동기화 안 됨
    @Override
    public SynchronizedList clone() {
        SynchronizedList result = (SynchronizedList) super.clone();
        result.data = new ArrayList<>(data);
        return result;
    }
    
    // ✅ 동기화 적용
    @Override
    public synchronized SynchronizedList clone() {
        try {
            SynchronizedList result = (SynchronizedList) super.clone();
            result.data = new ArrayList<>(data);
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

---

### ⚠️ 4. 상속용 클래스 설계

**방법 1: Object 방식 모방**
```java
public class Parent implements Cloneable {
    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
// 하위 클래스가 Cloneable 구현 여부 선택 가능
```

**방법 2: clone 비활성화**
```java
public class Parent {
    @Override
    protected final Object clone() throws CloneNotSupportedException {
        throw new CloneNotSupportedException();
    }
}
// 하위 클래스에서 재정의 불가
```

---

## 📌 6. 더 나은 대안: 복사 생성자/팩터리

### 복사 생성자
```java
public class Yoshi {
    private String name;
    private List<String> items;
    
    // 복사 생성자
    public Yoshi(Yoshi original) {
        this.name = original.name;
        this.items = new ArrayList<>(original.items);  // 깊은 복사
    }
}

// 사용
Yoshi original = new Yoshi("Mario");
Yoshi copy = new Yoshi(original);  // 명확하고 간단!
```

### 복사 팩터리
```java
public class Yoshi {
    private String name;
    private List<String> items;
    
    // 복사 팩터리
    public static Yoshi newInstance(Yoshi original) {
        Yoshi copy = new Yoshi();
        copy.name = original.name;
        copy.items = new ArrayList<>(original.items);
        return copy;
    }
}

// 사용
Yoshi copy = Yoshi.newInstance(original);
```

---

### 복사 생성자/팩터리의 장점

#### 1. 자연스러운 객체 생성
```java
// clone: 생성자 우회
PhoneNumber copy = original.clone();

// 복사 생성자: 정상적인 생성
PhoneNumber copy = new PhoneNumber(original);
```

#### 2. final 필드 사용 가능
```java
public class Stack {
    private final Object[] elements;  // final 가능!
    
    public Stack(Stack original) {
        this.elements = original.elements.clone();  // 생성자에서 할당
    }
}
```

#### 3. 예외 처리 불필요
```java
// clone: 검사 예외 처리 필요
try {
    return (Phone) super.clone();
} catch (CloneNotSupportedException e) {
    throw new AssertionError();
}

// 복사 생성자: 예외 없음
public Phone(Phone original) {
    this.number = original.number;
}
```

#### 4. 형변환 불필요
```java
// clone: 형변환 필요
PhoneNumber copy = (PhoneNumber) original.clone();

// 복사 생성자: 형변환 없음
PhoneNumber copy = new PhoneNumber(original);
```

#### 5. 인터페이스 기반 복사 가능 (변환 생성자)
```java
// HashSet을 TreeSet으로 변환
public TreeSet(Collection<E> c) {
    // ...
}

HashSet<String> hashSet = new HashSet<>();
hashSet.add("apple");
hashSet.add("banana");

// 정렬된 TreeSet으로 변환!
TreeSet<String> treeSet = new TreeSet<>(hashSet);
```

**실전 예시:**
```java
// ArrayList를 LinkedList로
List<String> arrayList = new ArrayList<>();
List<String> linkedList = new LinkedList<>(arrayList);

// HashMap을 TreeMap으로
Map<String, Integer> hashMap = new HashMap<>();
Map<String, Integer> treeMap = new TreeMap<>(hashMap);
```

---

## 📌 7. 최종 정리 및 권장사항

### 언제 clone()을 사용해야 하나?

| 상황 | 권장 방법 | 이유 |
|------|----------|------|
| 배열 복사 | `array.clone()` | 가장 깔끔하고 효율적 |
| 이미 Cloneable 구현한 클래스 확장 | 어쩔 수 없이 `clone()` 구현 | 상속 구조 때문에 |
| 새로운 클래스 설계 | 복사 생성자/팩터리 | 안전하고 유연함 |
| final 클래스 + 성능 중요 | 신중히 검토 후 `clone()` | 위험 적지만 검증 필수 |

### 구현 체크리스트

```java
// ✅ 완벽한 clone() 구현 템플릿
public class MyClass implements Cloneable {
    private int primitiveField;           // 기본 타입
    private String immutableField;        // 불변 객체
    private int[] mutableArray;          // 가변 배열
    private List<String> mutableList;    // 가변 컬렉션
    
    @Override
    public MyClass clone() {
        try {
            // 1. super.clone() 호출
            MyClass result = (MyClass) super.clone();
            
            // 2. 가변 객체 깊은 복사
            result.mutableArray = mutableArray.clone();
            result.mutableList = new ArrayList<>(mutableList);
            
            // 3. 일련번호/고유ID가 있다면 새로 생성
            // result.id = generateNewId();
            
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
    
    @Override
    public boolean equals(Object obj) {
        // equals도 함께 구현
    }
    
    @Override
    public int hashCode() {
        // hashCode도 함께 구현
    }
}
```

### 핵심 원칙

1. **새 인터페이스/클래스: Cloneable 금지**
2. **배열 복사: clone() 사용 OK**
3. **일반적인 복사: 복사 생성자/팩터리 사용**
4. **불가피한 경우: 완벽하게 구현**

---

## 💡 실전 예제 비교

```java
// ❌ Cloneable 방식
public class Member implements Cloneable {
    private String name;
    private List<String> hobbies;
    
    @Override
    public Member clone() {
        try {
            Member result = (Member) super.clone();
            result.hobbies = new ArrayList<>(hobbies);
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}

// ✅ 복사 생성자 방식 (권장!)
public class Member {
    private final String name;           // final 가능!
    private final List<String> hobbies;  // final 가능!
    
    // 복사 생성자
    public Member(Member original) {
        this.name = original.name;
        this.hobbies = new ArrayList<>(original.hobbies);
    }
    
    // 변환 생성자 (더 유연함!)
    public Member(String name, Collection<String> hobbies) {
        this.name = name;
        this.hobbies = new ArrayList<>(hobbies);
    }
}

// 사용 예시
Member original = new Member("철수");
Member copy = new Member(original);  // 명확!

Set<String> hobbySet = new HashSet<>(Arrays.asList("독서", "운동"));
Member fromSet = new Member("영희", hobbySet);  // Set을 List로 변환!
```
---------------

Q&A

## 문제

다음 코드를 보고 **가장 적절한 설명**을 고르세요.

```java
public class StudentGroup implements Cloneable {
    private String groupName;
    private Student[] students;
    
    @Override
    public StudentGroup clone() {
        try {
            StudentGroup result = (StudentGroup) super.clone();
            result.students = students.clone();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}

class Student {
    private String name;
    private List<String> courses;
    
    public void addCourse(String course) {
        courses.add(course);
    }
}

// 사용 코드
StudentGroup original = new StudentGroup("A반");
StudentGroup copy = original.clone();

copy.students[0].addCourse("수학");
```

### 보기

**① 완벽한 깊은 복사이므로, 복제본을 수정해도 원본에 영향을 주지 않는다.**

**② students 배열만 복사되고 Student 객체는 공유되므로, 복제본 수정 시 원본도 영향을 받는다.**

**③ clone() 메서드가 public이 아니므로 컴파일 에러가 발생한다.**

**④ Student 클래스가 Cloneable을 구현하지 않았으므로 런타임 에러가 발생한다.**
