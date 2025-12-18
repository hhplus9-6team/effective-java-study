# item65. 리플렉션보다는 인터페이스를 사용하라.

## 리플렉션이 뭔데?

리플렉션은 쉽게 말해서 "런타임에 클래스 정보를 까보는 기능"이다.

보통 자바에서는 컴파일할 때 어떤 클래스를 쓸지 다 정해져 있어야 한다. 그런데 리플렉션을 쓰면 프로그램이 실행되는 도중에 "아, 이 클래스가 뭔지 모르겠는데 일단 열어서 생성자랑 메서드 다 꺼내볼게"가 가능해진다.

```java
// 일반적인 방식: 컴파일 타임에 타입을 안다
Set<String> set = new HashSet<>();

// 리플렉션 방식: 런타임에 클래스 이름만 알면 된다
Class<?> clazz = Class.forName("java.util.HashSet");
Object set = clazz.getDeclaredConstructor().newInstance();
```

---

## 리플렉션으로 할 수 있는 것들

리플렉션의 능력을 정리하면 다음과 같다.

**Class 객체로 얻을 수 있는 것:**
- Constructor: 생성자 정보
- Method: 메서드 정보
- Field: 필드 정보

**이 정보들로 할 수 있는 것:**
- 멤버 이름, 필드 타입, 메서드 시그니처 조회
- 인스턴스 생성 (Constructor.newInstance())
- 메서드 호출 (Method.invoke())
- 필드 값 읽기/쓰기 (Field.get(), Field.set())

---

## 리플렉션의 3대 단점

근데 이 강력한 기능에는 심각한 단점이 있다.

```
┌─────────────────────────────────────────────────────────────┐
│                    리플렉션의 3대 단점                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 컴파일타임 타입 검사 불가능                               │
│     → 오타나 잘못된 메서드 호출을 컴파일러가 못 잡음           │
│     → 런타임에 터짐 (최악의 상황)                            │
│                                                             │
│  2. 코드가 지저분해짐                                        │
│     → try-catch 범벅                                        │
│     → 가독성 최악                                           │
│                                                             │
│  3. 성능 저하                                               │
│     → 일반 메서드 호출 대비 약 11배 느림                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 단점 1번을 코드로 보자

일반 코드 vs 리플렉션 코드를 비교해보면 확 와닿는다.

```java
// 일반 코드: 컴파일 타임에 오류 발견
Set<String> set = new HashSet<>();
set.addAll(List.of("a", "b"));  // 컴파일러가 타입 체크 해줌
set.nonExistentMethod();        // 컴파일 에러! 바로 발견

// 리플렉션 코드: 런타임에 터짐
Class<?> clazz = Class.forName("java.util.HashSet");
Object set = clazz.getDeclaredConstructor().newInstance();
Method m = clazz.getMethod("nonExistentMethod");  // 런타임 에러!
```

컴파일러가 잡아줘야 할 에러를 사용자가 직접 겪게 되는 것이다. 진짜 최악이다.

---

## 단점 2번: 예외 처리 지옥

리플렉션 쓰면 예외 처리가 미친듯이 늘어난다.

```java
// 리플렉션 없이: 한 줄
Set<String> set = new HashSet<>();

// 리플렉션으로: 예외만 6종류
try {
    Class<?> cl = Class.forName(className);           // ClassNotFoundException
    Constructor<?> cons = cl.getDeclaredConstructor(); // NoSuchMethodException
    Object obj = cons.newInstance();                   // IllegalAccessException
                                                       // InstantiationException
                                                       // InvocationTargetException
                                                       // ClassCastException
} catch (...) {
    // 예외 처리 코드 잔뜩...
}
```

**한 줄이면 될 걸 25줄 넘게 써야 한다.** 유지보수하는 사람 입장에선 지옥이다.

---

## 그럼 리플렉션은 언제 쓰나?

리플렉션이 꼭 필요한 경우가 있긴 하다.

**리플렉션이 필요한 경우:**
- 코드 분석 도구 (IDE의 자동완성 기능 등)
- 의존관계 주입 프레임워크 (Spring 같은)
- 플러그인 시스템 (런타임에 플러그인 로드)

하지만 이런 프레임워크들조차 리플렉션 사용을 점점 줄이는 추세다. 단점이 너무 명백하기 때문이다.

---

## 핵심: 리플렉션은 인스턴스 생성에만 쓰고, 참조는 인터페이스로

이게 이번 아이템의 핵심이다. 흐름을 그림으로 보자.

```
┌──────────────────────────────────────────────────────────────────┐
│                    올바른 리플렉션 사용 패턴                       │
└──────────────────────────────────────────────────────────────────┘

    [컴파일 타임]                      [런타임]
         │                               │
         ▼                               ▼
┌─────────────────┐           ┌─────────────────────┐
│  Set<String>    │           │  "java.util.HashSet"│
│  (인터페이스)    │◄──참조────│  or                 │
│                 │           │  "java.util.TreeSet"│
└─────────────────┘           │  (문자열로 전달)     │
         │                    └─────────────────────┘
         │                               │
         │                               ▼
         │                    ┌─────────────────────┐
         │                    │  리플렉션으로       │
         │                    │  인스턴스 생성      │
         └────────────────────┤  Class.forName()   │
                              │  newInstance()     │
                              └─────────────────────┘

핵심 포인트:
- 리플렉션은 "객체 생성"에만 사용
- 생성된 객체는 "인터페이스 타입"으로 참조
- 이후 사용은 일반 코드와 동일
```

---

## 실제 코드로 보는 올바른 패턴

책의 예제 코드를 단계별로 뜯어보자.

### Step 1: 클래스 이름을 Class 객체로 변환

```java
// args[0]에 "java.util.HashSet" 또는 "java.util.TreeSet" 전달
Class<? extends Set<String>> cl = null;
try {
    cl = (Class<? extends Set<String>>) Class.forName(args[0]);
} catch (ClassNotFoundException e) {
    fatalError("클래스를 찾을 수 없습니다.");
}
```

여기서 `Class.forName()`은 문자열로 된 클래스 이름을 받아서 해당 Class 객체를 반환한다.

### Step 2: 생성자 얻기

```java
Constructor<? extends Set<String>> cons = null;
try {
    cons = cl.getDeclaredConstructor();  // 기본 생성자(매개변수 없는)를 가져옴
} catch (NoSuchMethodException e) {
    fatalError("매개변수 없는 생성자를 찾을 수 없습니다.");
}
```

### Step 3: 인스턴스 생성 (리플렉션은 여기까지만!)

```java
Set<String> s = null;  // 인터페이스 타입으로 선언!
try {
    s = cons.newInstance();  // 여기서 실제 객체 생성
} catch (IllegalAccessException e) { ... }
  catch (InstantiationException e) { ... }
  catch (InvocationTargetException e) { ... }
  catch (ClassCastException e) { ... }
```

### Step 4: 이후는 일반 코드처럼 사용

```java
// 여기부터는 리플렉션이 아니다! 그냥 Set 인터페이스 메서드 호출
s.addAll(Arrays.asList(args).subList(1, args.length));
System.out.println(s);
```

---

## 이 패턴의 장점

```
┌────────────────────────────────────────────────────────────┐
│  리플렉션 영역 (생성만)     │  일반 코드 영역 (사용)       │
├────────────────────────────────────────────────────────────┤
│                            │                              │
│  - 예외 처리 필요          │  - 타입 안전                 │
│  - 코드가 복잡함           │  - 컴파일 타임 검사 가능     │
│  - 객체 생성에만 한정      │  - 코드 간결                 │
│                            │  - 성능 좋음                 │
│                            │                              │
└────────────────────────────────────────────────────────────┘
          ↑                              ↑
        최소화                         대부분의 코드
```

리플렉션의 단점이 "객체 생성 부분에만 국한"되고, 나머지 코드는 깔끔하게 유지된다.

---

## 실행 예시

```bash
# HashSet 사용 (순서 보장 안됨)
$ java ReflectiveSet java.util.HashSet a b c a b
[b, c, a]

# TreeSet 사용 (정렬됨)  
$ java ReflectiveSet java.util.TreeSet a b c a b
[a, b, c]
```

같은 코드인데 런타임에 전달하는 클래스 이름만 바꾸면 동작이 달라진다. 이게 리플렉션의 강력함이다.

---

## 리플렉션 예외 처리 팁

자바 7부터는 모든 리플렉션 예외의 공통 부모인 `ReflectiveOperationException`을 쓸 수 있다.

```java
// Before: 예외마다 catch
try {
    ...
} catch (ClassNotFoundException e) { ... }
  catch (NoSuchMethodException e) { ... }
  catch (IllegalAccessException e) { ... }
  // ... 계속

// After: 한 번에 처리
try {
    ...
} catch (ReflectiveOperationException e) {
    fatalError("리플렉션 오류: " + e.getMessage());
}
```

코드가 좀 더 간결해진다.

---

## 리플렉션의 또 다른 쓰임: 버전 호환성

외부 라이브러리가 버전별로 API가 다를 때 유용하다.

```
┌─────────────────────────────────────────────────────────────┐
│               버전 호환성을 위한 리플렉션 사용                 │
└─────────────────────────────────────────────────────────────┘

  컴파일: 가장 오래된 버전(v1.0) 기준으로 빌드
  
  런타임:
  ┌─────────────┐
  │  v1.0 환경  │ → 기본 기능만 동작
  └─────────────┘
  
  ┌─────────────┐
  │  v2.0 환경  │ → 리플렉션으로 새 API 확인
  └─────────────┘   → 있으면 사용, 없으면 대체 수단 사용
```

이렇게 하면 하나의 코드로 여러 버전을 지원할 수 있다.

---

## 핵심 정리

```
┌─────────────────────────────────────────────────────────────────┐
│                         핵심 정리                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 리플렉션은 강력하지만 단점이 많다                            │
│     - 컴파일타임 타입 검사 불가                                  │
│     - 코드가 지저분해짐                                         │
│     - 성능 저하                                                 │
│                                                                 │
│  2. 꼭 써야 한다면 "생성에만" 사용하라                           │
│     - 리플렉션은 newInstance()까지만                            │
│     - 이후는 인터페이스나 상위 클래스로 참조                     │
│                                                                 │
│  3. 리플렉션이 필요한지 확신이 없다면?                           │
│     → 아마 필요 없을 가능성이 높다                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 이해도 테스트

**문제 1.** 리플렉션의 단점으로 틀린 것은?

A) 컴파일타임에 타입 검사를 받을 수 없다  
B) 코드가 길어지고 가독성이 떨어진다  
C) 일반 메서드 호출보다 성능이 느리다  
D) 런타임에 존재하지 않는 클래스는 사용할 수 없다

---

**문제 2.** 다음 중 리플렉션을 올바르게 사용한 패턴은?

A) 리플렉션으로 객체를 생성하고, 리플렉션으로 메서드를 호출한다  
B) 리플렉션으로 객체를 생성하고, 인터페이스 타입으로 참조하여 사용한다  
C) 인터페이스 타입으로 객체를 생성하고, 리플렉션으로 메서드를 호출한다  
D) 모든 클래스를 리플렉션으로 다루는 것이 유연성 측면에서 좋다

---

**문제 3.** 다음 코드에서 `s` 변수의 타입을 `Set<String>`이 아닌 `HashSet<String>`으로 선언하면 어떤 문제가 생기는가?

```java
Class<? extends Set<String>> cl = (Class<? extends Set<String>>) 
    Class.forName(args[0]);  // args[0]이 "java.util.TreeSet"일 수도 있음
Constructor<? extends Set<String>> cons = cl.getDeclaredConstructor();
Set<String> s = cons.newInstance();  // 이 부분
```

A) 컴파일 에러가 발생한다  
B) TreeSet을 전달해도 정상 동작한다  
C) TreeSet을 전달하면 런타임에 ClassCastException이 발생한다  
D) 성능이 저하된다

