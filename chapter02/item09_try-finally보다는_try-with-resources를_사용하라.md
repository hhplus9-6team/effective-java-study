# item 09. try-finally보다는 try-with-resources를 사용하라
> - `try-finally`는 예외 누락, 중첩, 가독성 문제를 안고 있다.
> - `try-with-resources`는 **자바 7부터 제공되는 강력한 대안**으로, 예외 안전성과 코드 간결성을 모두 보장한다.
> - 닫아야 하는 자원을 다룰 땐 항상 **`try-with-resources`를 기본 선택지**로 사용하자.

---

## 1. 자원(Resource)과 close 문제
- 자바 라이브러리에는 **사용 후 닫아야 하는 자원**들이 많다.
  - 예: `InputStream`, `OutputStream`, `java.sql.Connection`, `Scanner`, `PrintWriter` 등
- 이런 자원은 사용 후 **`close()` 메서드를 호출해 명시적으로 닫아야** 한다.
- 하지만 클라이언트가 이를 놓치기 쉽고, 닫는 과정에서 예외가 발생할 경우 **자원이 제대로 해제되지 않아** 성능 저하, 자원 고갈 등의 문제가 발생할 수 있다.

---

## 2. 기존 방식: try-finally
자바 7 이전에는 자원을 닫기 위해 `try-finally`를 사용했다.

```java
static String firstLineOfFile(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br.readLine();
    } finally {
        br.close();
    }
}
````

### 문제점

1. **예외 누락 가능성**

   - `readLine()`이 예외를 던지고, `close()`에서도 예외가 던져지면,
      **두 번째 예외(`close`)가 첫 번째 예외(`readLine`)를 덮어써버림** → 디버깅 어려움

2. **코드가 지저분해짐**

   - 자원이 여러 개라면 `finally` 블록을 중첩해야 하므로 가독성이 떨어진다.

   ```java
   InputStream in = new FileInputStream(src);
   try {
       OutputStream out = new FileOutputStream(dst);
       try {
           // 파일 복사 로직
       } finally {
           out.close();
       }
   } finally {
       in.close();
   }
   ```

---

## 3. 개선된 방식: try-with-resources (Java 7 도입)

- `try-with-resources` 구문은 **`AutoCloseable` 인터페이스**를 구현한 자원을 자동으로 닫아준다.
- 사용법은 단순히 **`try` 블록에 자원을 선언**하면 된다.

```java
static String firstLineOfFile(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
}
```

- `try` 블록이 끝나면 `br.close()`가 자동으로 호출된다.
- 닫는 과정에서 예외가 발생해도, 원래 발생한 예외를 **보존**하면서 `close()` 예외는 *suppressed exception*으로 함께 기록된다.

---

## 4. try-with-resources의 장점

1. **예외가 누락되지 않는다**

   - 여러 예외가 발생해도 첫 번째 예외는 보존되고, 추가 예외는 **suppressed exception**으로 확인 가능하다.
   - `Throwable.getSuppressed()`로 추가 예외를 확인할 수 있다.

2. **코드가 깔끔해진다**

    - 자원이 여러 개여도 선언부에 나열만 하면 된다.

   ```java
   try (InputStream in = new FileInputStream(src);
        OutputStream out = new FileOutputStream(dst)) {
       // 파일 복사 로직
   }
   ```

3. **표준화된 자원 관리**

   - `AutoCloseable`을 구현하기만 하면, 사용자 정의 클래스도 `try-with-resources`에서 안전하게 사용할 수 있다.

---

## 5. AutoCloseable 인터페이스

- `try-with-resources`에 사용되는 모든 자원은 `AutoCloseable`을 구현해야 한다.
- `close()` 메서드 하나만 선언되어 있다.

```java
public interface AutoCloseable {
    void close() throws Exception;
}

```

### 사용자 정의 자원 예시

```java
class MyResource implements AutoCloseable {
    @Override
    public void close() {
        System.out.println("자원 해제");
    }

    public void work() {
        System.out.println("작업 수행");
    }
}

public static void main(String[] args) {
    try (MyResource res = new MyResource()) {
        res.work();
    }
}
```