# item08: finalizer와 cleaner 사용을 피하라

## 핵심 요약

### 1. finalizer와 cleaner를 피해야 하는 이유

**① 실행 시점을 보장할 수 없음**
```java
// ❌ 나쁜 예: finalizer에 파일 닫기를 맡김
public class BadFileHandler {
    private FileInputStream file;
    
    @Override
    protected void finalize() {
        file.close(); // 언제 실행될지 모름!
    }
}
```
→ 파일이 닫히지 않아 시스템의 파일 개수 제한에 걸려 프로그램이 멈출 수 있음

**잠깐! 시스템에서 제한된 파일갯수는 보통 몇개?**
- Windows 시스템
 - 기본 제한: 약 512개 (C 런타임 라이브러리 기준)
 - 최대 확장: 약 8192개까지 가능 (_setmaxstdio 함수 사용)

**② 성능 문제**
- 일반 객체: 12ns
- finalizer 사용: 550ns (약 50배 느림!)

**③ 예외 무시**
```java
@Override
protected void finalize() {
    throw new RuntimeException("오류!");
    // 이 예외는 무시되고, 경고도 출력되지 않음
}
```

**④ 보안 문제 (finalizer 공격)**
```java
public class VulnerableClass {
    public VulnerableClass() {
        if (!authorized) {
            throw new SecurityException("권한 없음!");
        }
    }
}

// 공격자가 만든 클래스
public class AttackClass extends VulnerableClass {
    @Override
    protected void finalize() {
        // 생성자에서 예외가 발생해도 finalizer는 실행됨!
        // 여기서 권한 없이 작업 수행 가능
    }
}
```

### 2. 올바른 대안: AutoCloseable

```java
// ✅ 좋은 예: try-with-resources 사용
public class GoodFileHandler implements AutoCloseable {
    private FileInputStream file;
    private boolean closed = false;
    
    @Override
    public void close() {
        if (closed) return;
        file.close();
        closed = true;
    }
    
    public void readFile() {
        if (closed) {
            throw new IllegalStateException("이미 닫힌 파일입니다!");
        }
        // 파일 읽기 작업
    }
}

// 사용
try (GoodFileHandler handler = new GoodFileHandler()) {
    handler.readFile();
} // 자동으로 close() 호출됨
```

### 3. cleaner를 쓸 수 있는 두 가지 경우

**① 안전망 역할** (close를 깜빡한 경우 대비)
```java
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();
    
    private final Cleaner.Cleanable cleanable;
    
    public Room() {
        State state = new State();
        this.cleanable = cleaner.register(this, state);
    }
    
    @Override
    public void close() {
        cleanable.clean(); // 명시적으로 청소
    }
    
    // Room을 참조하면 안 됨! (순환 참조 방지)
    private static class State implements Runnable {
        @Override
        public void run() {
            System.out.println("방 청소");
        }
    }
}

// 올바른 사용
try (Room room = new Room()) {
    System.out.println("방 사용");
} // "방 청소" 출력

// 잘못된 사용
new Room();
System.exit(0); // "방 청소" 출력 보장 안 됨!
```

**② 네이티브 피어 자원 회수**
- 네이티브 메서드로 생성된 C/C++ 객체는 GC가 모름
- 즉시 회수 필요 없고, 성능 저하 감당 가능한 경우만 사용

**잠깐! 네이티브 피어 자원이란?**
- 네이티브 피어는 자바 객체가 JNI(Java Native Interface)를 통해 C/C++로 작성된 네이티브 코드의 기능을 사용할 때, 그 네이티브 코드 측에서 생성된 객체를 말합니다.
- **네이티브 피어**: 자바가 C/C++ 코드를 사용할 때 생성되는 네이티브 메모리의 객체
- **문제점**: GC가 자바 객체만 회수하고 네이티브 메모리는 모름
- **해결책**: `AutoCloseable`로 명시적 해제 (cleaner는 안전망으로만)
- **cleaner 사용 조건**: 성능 저하 감당 가능 + 자원이 중요하지 않을 때만

## 결론
- **기본 원칙**: finalizer와 cleaner는 사용하지 말 것
- **대신 사용**: `AutoCloseable` + `try-with-resources`
- **예외적 사용**: 안전망 역할 또는 중요하지 않은 네이티브 자원 회수
- **주의사항**: 사용하더라도 불확실성과 성능 저하 감수해야 함