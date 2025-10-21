# item24. 멤버 클래스는 되도록 static으로 만들라

> [!IMPORTANT]
> 멤버 클래스에서 외부 인스턴스에 접근할 일이 없다면 무조건 `static`으로 정적 멤버 클래스로 만들자.

## 중첩 클래스의 종류


| 종류 | 설명 |
|---|---|
| **정적 멤버 클래스** | 바깥 클래스의 인스턴스와 무관하게 존재하는 클래스 |
| **비정적 멤버 클래스** | 바깥 클래스 인스턴스와 암묵적으로 연결된 클래스 |
| **익명 클래스** | 이름이 없고 선언과 동시에 인스턴스화되는 일회성 클래스 |
| **지역 클래스** | 메서드/블록 안에 선언되는 이름 있는 클래스 (유효 범위는 지역 변수와 동일) |

> - 멤버 클래스는 클래스 안에 선언된 클래스를 의미
> - 정적 멤버 클래스 외에 다 내부 클래스에 해당

### 정적 멤버 클래스

- **정의**: 바깥 클래스 안에 선언되며, 바깥 클래스의 **정적 멤버**에만 접근 가능
- **특징**
    - 바깥 인스턴스와 **독립적**으로 존재
    - `private` 정적 멤버 포함 바깥 클래스의 정적 멤버 접근 가능
    - 바깥 **인스턴스 필드에는 접근 불가**

```java
public class Outer {
    
    private static String staticField = "static field";
    private String instanceField = "instance field";

    public static class StaticMemberClass {
        public void display() {
            System.out.println(staticField);
            // System.out.println(instanceField);    // 컴파일 오류: 인스턴스 멤버 접근 불가
        }
    }
}
```
- **접근 제한**: `private`으로 선언하면 바깥 클래스 내부에서만 사용 가능

### 비정적 멤버 클래스

- **정의**: `static` 키워드 없이 선언된 멤버 클래스
- **특징**
    - 바깥 클래스 인스턴스와 **암묵적으로 연결**
    - 바깥 클래스의 **모든 멤버(인스턴스 필드 포함)** 접근 가능
    - 인스턴스 생성 시 **반드시 바깥 인스턴스가 필요**
    - 바깥 인스턴스 멤버 접근 시 **정규화된 this** 사용 가능: `Outer.this.someField`

```java
public class Outer {
    private String instanceField = "instance field";
    
    public class NonStaticMemberClass {
        public void display() {
            System.out.println(instanceField); // 바깥 인스턴스 멤버
            System.out.println(Outer.this.instanceField); // 정규화된 this로도 접근 가능
        }
    }
}
```

### 정적 멤버 클래스 vs 비정적 맴버클래스

| 구분 | 정적 멤버 클래스 | 비정적 멤버 클래스 |
|---|---|---|
| 선언 | `static class Inner {}` | `class Inner {}` |
| 바깥 인스턴스 필요 | ❌ | ✅ (암묵적 참조 유지) |
| 접근 가능 멤버 | 바깥 클래스 **정적** 멤버 | 바깥 클래스 **모든** 멤버 |
| 사용 목적 | 바깥 인스턴스와 **독립** 동작 | 바깥 인스턴스 **상태**를 참조/조작해야 함 |


### ⚠️ 비정적 멤버 클래스의 문제점

1. **숨은 외부 참조**  
   `static`을 생략하면 바깥 인스턴스에 대한 **암묵적 참조**가 생김
2. **메모리 누수 가능성**  
   비정적 멤버 클래스 인스턴스가 오래 살아남으면, 암묵적 참조 때문에 바깥 인스턴스가 **GC되지 못할 수 있음**

    <details><summary>예시</summary>
    🔴 문제 상황: 비정적 멤버 클래스로 인한 메모리 누수
    
    ```java
    public class OuterClass {
        private byte[] data = new byte[1024 * 1024]; // 1MB
        
        // 비정적 멤버 클래스 (static 없음)
        public class InnerTask {
            public void doSomething() {
                System.out.println("작업 수행");
            }
        }
        
        public InnerTask createTask() {
            return new InnerTask();
        }
    }
    
    // 사용 예시
    public class MemoryLeakExample {
        private static List<Object> taskCache = new ArrayList<>();
        
        public static void main(String[] args) {
            for (int i = 0; i < 10; i++) {
                OuterClass outer = new OuterClass(); // 1MB 객체 생성
                
                // InnerTask만 캐시에 저장
                taskCache.add(outer.createTask());
                
                // outer는 더 이상 사용하지 않음
                // 하지만 GC되지 않음!
            }
            
            // taskCache가 InnerTask를 참조
            // → InnerTask가 OuterClass를 암묵적으로 참조 (숨은 참조)
            // → OuterClass(1MB)가 GC되지 못함
            // 결과: 10MB 메모리 누수!
        }
    }
    ```
        taskCache (static 필드 = GC Root)
        │
        ├─> InnerTask#1
        │      └─> this$0 ──> OuterClass#1 (1MB) 💀 GC 불가
        │
        ├─> InnerTask#2  
        │      └─> this$0 ──> OuterClass#2 (1MB) 💀 GC 불가
        │
        └─> ... (10개)
    
        👉 InnerTask가 살아있는 한, OuterClass도 GC 안됨!
    
    🟢 해결책: 정적 멤버 클래스 사용
    ```java
    public class OuterClass {
    private byte[] data = new byte[1024 * 1024]; // 1MB
    
        // 정적 멤버 클래스
        public static class InnerTask {
            public void doSomething() {
                System.out.println("작업 수행");
            }
        }
        
        public InnerTask createTask() {
            return new InnerTask();
        }
    }
    
    // 사용 예시
    public class NoMemoryLeakExample {
        private static List<Object> taskCache = new ArrayList<>();
    
        public static void main(String[] args) {
            for (int i = 0; i < 10; i++) {
                OuterClass outer = new OuterClass(); // 1MB 객체 생성
                
                taskCache.add(outer.createTask());
                
                // outer는 GC됨!
                // InnerTask가 outer를 참조하지 않기 때문
            }
            
            // taskCache는 InnerTask만 보유
            // OuterClass는 모두 GC됨
            // 메모리: 최소화!
        }
    }
    ```
    </details>


3. **디버깅 어려움**  
   참조가 코드에 **명시적으로 드러나지 않아** 원인 추적이 어려움

---

### 익명 클래스
- 이름 없음
- 쓰이는 시점에 **선언과 동시에 인스턴스화**
- **일회성** 용도에 적합
- **비정적 문맥**에서만 바깥 인스턴스 참조 가능
    ```java
    public class Outer {
        private int outerField = 10;
    
        public void nonStaticMethod() {
            // 비정적 메서드 안에서 익명 클래스 생성
            Runnable r = new Runnable() {
                @Override
                public void run() {
                    // ✅ 바깥 인스턴스의 필드에 접근 가능
                    System.out.println(outerField);
                    // ✅ 바깥 인스턴스의 메서드 호출 가능
                    someInstanceMethod();
                }
            };
        }
    
        private void someInstanceMethod() { }
    }
    ```
- 정적 문맥에서는 **상수용 `final` 기본타입/문자열**만 가질 수 있음
  - 정적 메서드는 인스턴스와 무관하게 동작, 따라서 익명 클래스도 바깥 인스턴스를 참조할 수 없음
    ```java
    public class Outer {
        private int outerField = 10;
        
        public static void staticMethod() {
            final int localConst = 20;
            final String text = "Hello";
            int normalVar = 30;
            
            Runnable r = new Runnable() {
                @Override
                public void run() {
                    // ❌ 컴파일 에러: 정적 문맥이므로 인스턴스 필드 접근 불가
                    // System.out.println(outerField);
                    
                    // ✅ final 기본타입/문자열 상수는 사용 가능
                    System.out.println(localConst);
                    System.out.println(text);
                    
                    // ✅ Java 8+ : effectively final 변수도 가능
                    System.out.println(normalVar);
                }
            };
        }
    }
    ```
- `instanceof`나 **클래스 이름**이 필요한 작업 불가
- **여러 인터페이스 동시 구현 불가**, 상속과 구현을 동시에 다른 타입으로 할 수도 없음
- 상위 타입의 멤버 외에는 호출 불가
- 표현식 중간에 등장하므로 **길어지면 가독성 저하**
- 람다 도입 전에는 즉석 함수/처리 객체나 정적 팩터리 구현 등에 자주 사용

### 지역 클래스
- 메서드/블록 등 **지역 변수 선언 가능**한 곳이면 어디든 선언 가능
- **유효 범위**는 지역 변수와 동일
- 이름이 있어 **반복 사용 가능**
- **비정적 문맥**에서 바깥 인스턴스 참조 가능
- 익명 클래스로 표현하기 어려운 경우 대안

## 질문
Q. 멤버 클래스를 `static`으로 선언하지 않으면 어떤 문제가 발생하나요?
