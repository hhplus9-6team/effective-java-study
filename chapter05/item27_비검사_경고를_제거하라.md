# item 27: 비검사 경고를 제거하라

## 📌 핵심 개념

제네릭을 사용하면 컴파일러가 다양한 **비검사 경고(unchecked warning)**를 발생시킵니다. 이런 경고들은 가능한 한 모두 제거해야 타입 안전성을 보장할 수 있습니다.

---

## 🔍 비검사 경고의 종류

###  1. 비검사 형변환 경고 (Unchecked Cast)
```java
// ❌ 경고 발생
Object obj = "Hello";
String str = (String) obj;  // OK - 일반 형변환

List list = new ArrayList();
list.add("Hello");
List<String> stringList = (List<String>) list;  // ⚠️ 비검사 형변환 경고!
```
**경고 메시지:**
```
warning: [unchecked] unchecked cast
required: List<String>
found: List
```

###  2. 비검사 메서드 호출 경고 (Unchecked Method Invocation)
```java
// ❌ 경고 발생
List rawList = new ArrayList();
rawList.add("Hello");  // ⚠️ 비검사 메서드 호출 경고!

// ✅ 올바른 방법
List<String> typedList = new ArrayList<>();
typedList.add("Hello");  // OK
```
**경고 메시지:**
```
warning: [unchecked] unchecked call to add(E)
```

###  3. 검사 매개변수화 가변인수 타입 경고 (Unchecked Generic Array Creation)
```java
// ❌ 경고 발생
public static <T> void printAll(T... args) {  // ⚠️ 비검사 가변인수 경고!
    for (T arg : args) {
        System.out.println(arg);
    }
}

// 사용
List<String> list1 = Arrays.asList("A");
List<String> list2 = Arrays.asList("B");
printAll(list1, list2);  // ⚠️ 힙 오염 가능성!
```
**경고 메시지:**
```
warning: [unchecked] Possible heap pollution from parameterized vararg type
```

**✅ 안전하다면 @SafeVarargs 사용:**
```java
@SafeVarargs
public static <T> void printAll(T... args) {
    for (T arg : args) {
        System.out.println(arg);
    }
}
```

### 4. 비검사 변환 경고 (Unchecked Conversion)
```java
// ❌ 경고 발생
Set<String> names = new HashSet();  // ⚠️ 비검사 변환 경고!

// ✅ 올바른 방법
Set<String> names = new HashSet<>();  // 다이아몬드 연산자 사용
```
**경고 메시지:**
```
warning: [unchecked] unchecked conversion
required: Set<String>
found: HashSet
```

---

## 힙 오염(Heap Pollution)이란?

### 개념 설명
**힙 오염**은 매개변수화 타입의 변수가 **그 타입이 아닌 객체를 참조**할 때 발생합니다. 이는 주로 제네릭 가변인수(varargs)를 사용할 때 발생합니다.

### 왜 위험한가?
제네릭은 **컴파일 타임**에만 타입을 체크하고, **런타임**에는 **타입 소거(Type Erasure)** 때문에 타입 정보가 사라집니다. 이로 인해 힙 오염이 발생하면 **ClassCastException**이 예상치 못한 곳에서 터질 수 있습니다.

---

## 힙 오염 예시

### 예시 1: 위험한 제네릭 가변인수
```java
public class HeapPollutionExample {
    // ⚠️ 위험한 메서드 - 힙 오염 가능!
    static void dangerous(List<String>... stringLists) {
        List<Integer> intList = Arrays.asList(42);
        Object[] objects = stringLists;  // 배열 공변성으로 가능
        objects[0] = intList;             // ⚠️ 힙 오염 발생!
        String s = stringLists[0].get(0); // 💥 ClassCastException!
    }
    
    public static void main(String[] args) {
        List<String> list = Arrays.asList("Hello");
        dangerous(list);  // 런타임 에러 발생!
    }
}
```

**무슨 일이 일어났나?**
1. `List<String>...`은 내부적으로 `List[]` 배열로 변환됨
2. 배열의 공변성으로 `Object[]`로 참조 가능
3. `List<Integer>`를 `Object[]`에 넣음 → 힙 오염!
4. `List<String>`으로 접근하려 할 때 `ClassCastException` 발생

---

### 예시 2: 안전한 경우
```java
public class SafeVarargsExample {
    // ✅ 안전한 메서드 - 읽기만 함
    @SafeVarargs
    static <T> List<T> flatten(List<T>... lists) {
        List<T> result = new ArrayList<>();
        for (List<T> list : lists) {
            result.addAll(list);  // 읽기만 하므로 안전
        }
        return result;
    }
    
    public static void main(String[] args) {
        List<String> list1 = Arrays.asList("A", "B");
        List<String> list2 = Arrays.asList("C", "D");
        List<String> merged = flatten(list1, list2);
        System.out.println(merged);  // [A, B, C, D]
    }
}
```

**왜 안전한가?**
- 가변인수 배열을 **읽기만** 하고 수정하지 않음
- 가변인수 배열의 참조를 외부로 노출하지 않음

---

## 🛡️ 힙 오염을 피하는 방법

### 1. @SafeVarargs 사용 조건
메서드가 다음 조건을 **모두** 만족할 때만 `@SafeVarargs`를 사용하세요:

```java
@SafeVarargs
static <T> void safe(T... args) {
    // ✅ 1. 가변인수 배열에 아무것도 저장하지 않음
    // ✅ 2. 배열의 참조를 외부로 노출하지 않음
    // ✅ 3. 배열 요소를 읽기만 함
    
    for (T arg : args) {
        System.out.println(arg);  // 읽기만 하므로 안전
    }
}
```

### 2. 위험한 패턴
```java
// ❌ 위험 1: 가변인수 배열에 저장
static <T> void unsafe1(T... args) {
    args[0] = args[1];  // ❌ 배열 수정 - 위험!
}

// ❌ 위험 2: 배열 참조 노출
static <T> T[] unsafe2(T... args) {
    return args;  // ❌ 배열을 반환 - 위험!
}

// ❌ 위험 3: 다른 메서드로 전달
static <T> void unsafe3(T... args) {
    otherMethod(args);  // ❌ 배열을 다른 메서드로 전달 - 위험!
}
```

### 3. 안전한 대안: List 사용
```java
// ✅ 가변인수 대신 List를 사용
static <T> void safePrint(List<T> list) {
    for (T item : list) {
        System.out.println(item);
    }
}

// 사용
safePrint(Arrays.asList("A", "B", "C"));
```

---

## 💡 쉬운 예시로 이해하기

### ❌ 잘못된 코드
```java
Set<String> names = new HashSet();  // 경고 발생!
```

**컴파일러 경고:**
```
warning: [unchecked] unchecked conversion
required: Set<String>
found: HashSet
```

### ✅ 올바른 코드
```java
Set<String> names = new HashSet<>();  // 다이아몬드 연산자 사용
```

---

## 🎯 @SuppressWarnings 사용 원칙

### 1. 가능한 한 좁은 범위에 적용

#### ❌ 나쁜 예 - 메서드 전체에 적용
```java
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    if (a.length < size) {
        return (T[]) Arrays.copyOf(elements, size, a.getClass());
    }
    // ... 나머지 코드
    return a;
}
```

#### ✅ 좋은 예 - 지역변수에만 적용
```java
public <T> T[] toArray(T[] a) {
    if (a.length < size) {
        // 생성한 배열과 매개변수로 받은 배열의 타입이 모두 T[]로 같으므로
        // 올바른 형변환임
        @SuppressWarnings("unchecked") 
        T[] result = (T[]) Arrays.copyOf(elements, size, a.getClass());
        return result;
    }
    System.arraycopy(elements, 0, a, 0, size);
    if (a.length > size) a[size] = null;
    return a;
}
```

### 2. 반드시 주석으로 이유 설명

```java
@SuppressWarnings("unchecked")  // 배열 타입이 T[]로 동일하여 안전함
T[] result = (T[]) Arrays.copyOf(...);
```

---

## ⚠️ 주의사항

### 🚫 하지 말아야 할 것
- 클래스 전체에 `@SuppressWarnings` 적용
- 타입 안전성을 검증하지 않고 경고 숨기기
- 경고를 무시한 이유를 주석으로 남기지 않기

### ✅ 해야 할 것
- 가능한 모든 비검사 경고 제거
- 제거할 수 없다면 타입 안전성 검증 후 최소 범위에만 적용
- 경고를 숨긴 이유를 반드시 주석으로 작성

---

## 🎓 실전 예시

```java
public class Box<T> {
    private Object[] items = new Object[10];
    
    // ❌ 나쁜 예
    @SuppressWarnings("unchecked")
    public T get(int index) {
        return (T) items[index];  // 전체 메서드에 경고 숨김
    }
    
    // ✅ 좋은 예
    public T get(int index) {
        // Object[]에서 T로의 형변환은 제네릭 배열의 한계로 인해 불가피함
        // 하지만 items에는 T타입만 저장되므로 안전함
        @SuppressWarnings("unchecked")
        T item = (T) items[index];
        return item;
    }
}
```

---

## 핵심 정리

1. **모든 비검사 경고는 제거하라** - 타입 안전성 보장
2. **제거할 수 없다면** - 타입 안전성을 증명하고 최소 범위에 `@SuppressWarnings` 적용
3. **항상 주석을 남겨라** - 경고를 숨긴 이유를 명확히 설명
4. **힙 오염 주의** - 제네릭 가변인수 사용 시 안전성을 반드시 검증하고 `@SafeVarargs` 사용

---

##  이해도 Q&A 

**다음 중 `@SuppressWarnings("unchecked")` 애너테이션 사용에 대한 설명으로 가장 적절한 것은?**

**A)** 비검사 경고가 발생하면 즉시 클래스 전체에 `@SuppressWarnings`를 적용하여 경고를 제거하는 것이 좋다.

**B)** `@SuppressWarnings`를 사용할 때는 가능한 한 좁은 범위(지역변수, 짧은 메서드)에 적용하고, 반드시 경고를 숨긴 이유를 주석으로 남겨야 한다.

**C)** 비검사 경고는 대부분 무시해도 되는 경고이므로 타입 안전성 검증 없이 `@SuppressWarnings`를 사용해도 문제없다.

**D)** return 문에서 발생하는 비검사 경고는 제거할 방법이 없으므로 항상 메서드 전체에 `@SuppressWarnings`를 적용해야 한다.

