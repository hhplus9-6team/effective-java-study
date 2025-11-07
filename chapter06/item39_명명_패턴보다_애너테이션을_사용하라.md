# item39. 명명 패턴보다 애너테이션을 사용하라

> [!IMPORTANT]
> 테스트 메서드를 명명 패턴(testXXX) 대신 애너테이션(@Test)으로 표시하여 오타 위험을 없애고 타입 안전성과 가독성을 높인다.
> 또한 테스트를 넘어, 직접 도구를 만든다면 애너테이션을 함께 설계하고, 자바 표준 및 정적 분석용 애너테이션을 적극 활용하라.

## 문제 상황: 명명 패턴의 한계

자바 초기 JUnit 3에서는 테스트 메서드나 특정 동작을 이름으로 구분하는 경우가 많았다.  
예) 테스트 메서드 네이밍: `testXXX()`

명명 패턴 단점
- 오타로 인해 테스트가 누락될 수 있다 (예: tesXXX)
- <details>
  <summary>올바른 프로그램 요소(메서드 등)에서만 사용되리라 보증할 방법이 없다</summary>
  
  명명 패턴은 단순히 이름 문자열만 보기 때문에, 
  “메서드 단위만 인식”해야 하는 규칙을 언어 차원에서 보장할 수 없음  
  → “잘못된 위치(클래스, 필드 등)”에서도 패턴을 쓸 수 있다는 게 문제
  </details>
- <details>
  <summary>프로그램 요소(예: 예외 타입)를 매개변수로 전달할 마땅한 방법이 없다</summary>
  
  명명 패턴은 “문자열” 기반이라 프로그램 요소(클래스, 타입, 메서드 등)를 안전하게 전달할 방법이 없음  
  → ArithmeticException 같은 예외를 타입 안정적으로 넘길 수 없음
  </details>


## 해결책: 애너테이션 사용

> 애너테이션(Annotation)이란?
> 코드에 부가적인 정보를 붙이는 기능으로 컴파일러나 프레임워크(JUnit, Spring 등)가 인식해 특정 동작을 수행하도록 하는 역할을 함
> - **메타 애너테이션(Meta-annotation)**:  
    애너테이션을 정의할 때 사용하는 애너테이션  
    예: `@Retention`, `@Target`
> - **마커 애너테이션(Marker annotation)**:  
  요소(속성)가 없는 애너테이션으로, 존재 자체가 의미  
  예: `@Override`, `@Test`
> - **매개변수 애너테이션(Parameter annotation)**:  
  속성을 갖는 애너테이션으로, 값에 따라 다른 동작을 지정  
  예: `@ExceptionTest(ArithmeticException.class)`

JUnit 4부터는 **애너테이션** 기능을 도입

예시:
```java
@Retention(RetentionPolicy.RUNTIME) // 런타임에도 유지
@Target(ElementType.METHOD) // 메서드 선언시만 사용
public @interface Test { }
```

사용 예:
```java
public class Sample {
    @Test public static void m1() { }   // 성공
    public static void m2() { }         // 무시됨
    @Test public static void m3() { throw new RuntimeException("실패"); }
    @Test public void m3() { }          // 잘못 사용: 정적 메서드가 아님
}
```

따라서, 이제 테스트 프레임워크는 이름 대신 `@Test` 애너테이션이 붙은 메서드를 찾아 실행한다.


### 명명 패턴 vs 애너테이션

| 구분 | 명명 패턴 | 애너테이션 |
|------|------------|-------------|
| **인식 방식** | 문자열 패턴 | 메타데이터 |
| **오류 방지** | 오타에 취약 | 컴파일 시점 검증 가능 |
| **유연성** | 제한적 | 다양한 속성, 파라미터 추가 가능 |
| **가독성** | 규칙이 코드에 숨김 | 의도가 명확하게 드러남 |


## 확장: 예외를 기대하는 테스트

### 기본: 매개변수 하나를 받은 애너테이션 타입

예외 발생을 기대하는 테스트를 만들 때 애너테이션에 파라미터를 추가할 수 있다.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
    Class<? extends Throwable> value();
}
```

사용 예:
```java
public class Sample2 {
    @ExceptionTest(ArithmeticException.class)
    public static void m1() { int i = 0; i = i / i; }
}
```

이렇게 하면 테스트 프레임워크는 “이 테스트는 `ArithmeticException`이 발생해야 성공”이라는 정보를 명확하게 인식할 수 있다.

### 심화: 배열 매개변수를 받는 애너테이션 타입

#### 1. 배열 파라미터 방식
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTestMulti {
    Class<? extends Throwable>[] value();
}
```

사용 예:
```java
public class Sample3 {
    @ExceptionTestMulti({IndexOutOfBoundsException.class, 
            NullPointerException.class})
    public static void m1() {
        List<String> list = new ArrayList<>();
        list.add(null);
        list.get(1); // IOBE 또는 NPE 발생 가능
    }
}
```

#### 2. 반복 가능한 애너테이션(@Repeatable) 방식

- 자바 8에서부터 **반복 가능한 애너테이션**을 사용할 수 있다.  
- 하나의 애너테이션을 여러 번 선언할 수 있어, 코드 가독성과 선언의 유연성을 높인다.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Repeatable(ExceptionTests.class)
public @interface ExceptionTestRepeatable {
    Class<? extends Throwable> value();
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTestContainer {
    ExceptionTestRepeatable[] value();
}
```

사용 예:
```java
public class Sample4 {
    @ExceptionTestRepeatable(IndexOutOfBoundsException.class)
    @ExceptionTestRepeatable(NullPointerException.class)
    public static void m1() { throw new NullPointerException(); }
}
```

**설계 사항**
1. `@Repeatable` 를 단 애너테이션을 반환하는 **컨테이너 애너테이션을 반환**해야하고, 컨테이너 애너테이션의 class 객체를 매개변수로 전달해야한다.
2. 컨테이너 애너테이션은 **내부 애너테이션 타입의 배열을 반환하는 value 메서드를 정의**해야한다.
3. 컨테이너 애너테이션 타입에는 적절한 보존 정책(`@Retention`)과 적용 대상(`@Target`)을 명시해야한다(컴파일)

## 간단한 테스트 러너 구현 예
아래는 `@Test`, `@ExceptionTest`, `@ExceptionTestMulti`, `@ExceptionTestRepeatable`을 처리하는 미니 러너.  
핵심은 리플렉션으로 메서드를 순회하면서 isAnnotationPresent()로 애너테이션 존재 여부를 확인하고, 예외 기대 여부에 따라 테스트 결과를 분기하는 것이다.

하지만 **애너테이션을 선언하고 이를 처리하는 부분에서 코드 양이 늘어나면 처리가 복잡해져 오류가 날 가능성이 커짐을 명심하자.**

```java
import java.lang.reflect.*;

public class RunTests {
    public static void main(String[] args) throws Exception {
        int tests = 0;
        int passed = 0;
        Class<?> testClass = Class.forName(args[0]);

        for (Method m : testClass.getDeclaredMethods()) {
            // @Test (예외 기대 X)
            if (m.isAnnotationPresent(Test.class)) {
                tests++;
                try {
                    m.invoke(null);
                    passed++;
                } catch (InvocationTargetException wrappedEx) {
                    Throwable exc = wrappedEx.getCause();
                    System.out.printf("Test %s failed: %s %n", m, exc);
                } catch (Exception exc) {
                    System.out.printf("INVALID @Test: %s%n", m);
                }
            }

            // @ExceptionTest (단일 예외)
            if (m.isAnnotationPresent(ExceptionTest.class)) {
                tests++;
                Class<? extends Throwable> excType =
                        m.getAnnotation(ExceptionTest.class).value();
                try {
                    m.invoke(null);
                    System.out.printf("Test %s failed: no exception%n", m);
                } catch (InvocationTargetException wrappedEx) {
                    Throwable exc = wrappedEx.getCause();
                    if (excType.isInstance(exc))
                        passed++;
                    else
                        System.out.printf(
                                "Test %s failed: expected %s, got %s%n",
                                m, excType.getName(), exc);
                } catch (Exception exc) {
                    System.out.printf("INVALID @ExceptionTest: %s%n", m);
                }
            }

            // @ExceptionTest (배열 매개변수 버전)
            if (m.isAnnotationPresent(ExceptionTestMulti.class)) {
                tests++;
                Class<? extends Throwable>[] excTypes =
                        m.getAnnotation(ExceptionTestMulti.class).value();
                try {
                    m.invoke(null);
                    System.out.printf("Test %s failed: no exception%n", m);
                } catch (InvocationTargetException wrappedEx) {
                    Throwable exc = wrappedEx.getCause();
                    int oldPassed = passed;
                    for (Class<? extends Throwable> excType : excTypes) {
                        if (excType.isInstance(exc)) {
                            passed++;
                            break;
                        }
                    }
                    if (passed == oldPassed)
                        System.out.printf(
                                "Test %s failed: expected %s, got %s%n",
                                m, Arrays.toString(excTypes), exc);
                } catch (Exception exc) {
                    System.out.printf("INVALID @ExceptionTestMulti: %s%n", m);
                }
            }

            // @ExceptionTestRepeatable (반복 가능한 애너테이션 버전)
            if (m.isAnnotationPresent(ExceptionTestRepeatable.class)
                || m.isAnnotationPresent(ExceptionTestContainer.class)) {
                tests++;
                try {
                    m.invoke(null);
                    System.out.printf("Test %s failed: no exception%n", m);
                } catch (InvocationTargetException wrappedEx) {
                    Throwable exc = wrappedEx.getCause();
                    int oldPassed = passed;
                    ExceptionTestRepeatable[] excTests =
                            m.getAnnotationsByType(ExceptionTestRepeatable.class);
                    for (ExceptionTestRepeatable test : excTests) {
                        if (test.value().isInstance(exc)) {
                            passed++;
                            break;
                        }
                    }
                    if (passed == oldPassed)
                        System.out.printf(
                                "Test %s failed: expected one of %s, got %s%n",
                                m, Arrays.toString(excTests), exc);
                } catch (Exception exc) {
                    System.out.printf("INVALID @ExceptionTestRepeatable: %s%n", m);
                }
            }
        }

        System.out.printf("Passed: %d, Failed: %d%n", passed, tests - passed);
    }
}
```

### getAnnotationsByType() vs isAnnotationPresent()

| 메서드 | 설명 |
|---------|------|
| `isAnnotationPresent(Class<? extends Annotation> annotationClass)` | 해당 애너테이션이 선언되어 있는지 **존재 여부만 확인**한다. 반복 가능한 애너테이션이 여러 개여도 하나라도 있으면 true를 반환한다. |
| `getAnnotationsByType(Class<T> annotationClass)` | 지정한 타입의 애너테이션이 **여러 번 선언된 경우 모두 반환**한다. `@Repeatable`을 사용한 애너테이션 처리에 적합하다. |


## 추가 권장사항
1. **도구를 만든다면 적절한 애너테이션 타입도 함께 제공하라**  
   소스 코드에 추가 정보를 제공하는 도구를 만든다면, 그 정보를 명시적으로 표현할 수 있는 애너테이션을 정의하라.  
   예: `@Entity`, `@Controller`, `@Mapper` 등

2. **자바 표준 애너테이션은 반드시 사용하라**  
   자바가 기본 제공하는 애너테이션(`@Override`, `@Deprecated`, `@SuppressWarnings`, `@FunctionalInterface`)은  
   코드 품질을 높이고 컴파일러가 잘못된 사용을 잡아준다.

3. **정적 분석 도구 애너테이션을 적극 활용하라**  
   정적 분석 도구(FindBugs, Checker Framework, IntelliJ 등)가 제공하는  
   `@Nullable`, `@NonNull`, `@CheckReturnValue`, `@Immutable` 등의 애너테이션을 사용하면  
   코드의 진단 정보 품질을 향상시킬 수 있다.

## QnA
Q1. 명명 패턴 방식의 단점 3가지  
Q2. getAnnotationsByType()과 isAnnotationPresent()의 차이
