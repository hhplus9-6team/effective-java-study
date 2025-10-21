# ITEM22: 인터페이스는 타입을 정의하는 용도로만 사용하라

---

## 인터페이스

- 자신을 구현한 클래스의 인스턴스를 참조할 수 있는 타입 역할
=> 자신의 인스턴스로 무엇을 할 수 있는지를 얘기하는것

---

## 상수 인터페이스

### 상수 인터페이스 안티 패턴
```java
public interface PhysicalConstants {
    // 아보가드로 수 (1/몰)
    static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    
    // 볼츠만 상수 (J/K)
    static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    
    // 전자 질량 (kg)
    static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

### **왜 나쁜가?**

#### 1. **내부 구현을 API로 노출하는 문제**
클래스 내부에서 사용하는 상수는 외부 인터페이스가 아니라 **내부 구현**에 해당한다. 따라서 상수 인터페이스를 구현하는 것은 이 내부 구현을 클래스의 API로 노출하는 행위이다.

```java
// 상수 인터페이스 (내부 구현 세부사항)
public interface DatabaseConfig {
    String DB_URL = "jdbc:mysql://localhost:3306/mydb";
    String DB_USER = "root";
    String DB_PASSWORD = "password";
    int CONNECTION_TIMEOUT = 5000;
}

// UserService가 상수 인터페이스를 구현
public class UserService implements DatabaseConfig {
    // 내부에서만 사용할 DB 설정들
    public void saveUser(User user) {
        Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
        // ...
    }
}

// 클라이언트 코드
public class Main {
    public static void main(String[] args) {
        UserService service = new UserService();
        
        // 문제: 클라이언트가 내부 구현 세부사항에 접근 가능!
        String url = UserService.DB_URL;           // 내부 DB 설정 노출
        String user = UserService.DB_USER;         // 내부 DB 설정 노출
        String password = UserService.DB_PASSWORD; // 내부 DB 설정 노출
        int timeout = UserService.CONNECTION_TIMEOUT; // 내부 DB 설정 노출
        
        // 클라이언트에게는 전혀 의미 없는 정보들
    }
}
```

**문제점:**
- 클래스가 어떤 상수 인터페이스를 사용하든 사용자에게 아무런 의미가 없다
- 오히려 사용자에게 혼란을 주기도 한다
- 클라이언트 코드가 내부 구현에 해당하는 이 상수들에 종속되게 한다

#### 2. **바이너리 호환성 문제**
바이너리 호환성: 이미 컴파일된 클라이언트 코드가 이 상수 인터페이스를 참조하고 있기 때문에, 인터페이스를 제거하면 클라이언트 코드가 작동을 멈출 수 있습니다. 이는 클래스가 영원히 불필요한 상수에 묶이게 만듭니다.

```java
// 클라이언트가 이미 사용 중
public class ClientCode {
    public void connectToDatabase() {
        // UserService의 내부 상수를 직접 사용
        String url = UserService.DB_URL;
        String user = UserService.DB_USER;
        String password = UserService.DB_PASSWORD;
        
        // 이제 클라이언트가 내부 구현에 완전히 종속됨
        Connection conn = DriverManager.getConnection(url, user, password);
    }
}

// UserService 개발자가 "이제 다른 DB 설정을 쓰자"고 결정
public class UserService {
    // implements DatabaseConfig 제거하고 싶지만...
    // 클라이언트가 이미 DB_URL 등을 사용하고 있어서 제거 불가!
    
    // 새로운 DB 설정 사용
    private static final String NEW_DB_URL = "jdbc:postgresql://...";
    // 하지만 여전히 DatabaseConfig를 implements해야 함 (바이너리 호환성)
}
```

 - 클래스는 불필요한 상수에 영원히 묶이게 된다.

#### 3. **하위 클래스의 이름 공간 오염**
final이 아닌 클래스가 상수 인터페이스를 구현하면 모든 하위 클래스의 이름공간이 그 인터페이스가 정의한 상수들로 오염되어버린다.

```java
// 상수 인터페이스
public interface CookingConstants {
    String KNIFE = "칼";
    String PAN = "팬";
    String SPATULA = "주걱";
    String OVEN = "오븐";
    String MICROWAVE = "전자레인지";
}

// 부모 클래스 (요리사)
public class Chef implements CookingConstants {
    public void cook() {
        System.out.println("요리를 시작합니다");
        // 요리사에게는 필요한 상수들
    }
}

// 하위 클래스 1 (캐셔 - 요리와 무관)
public class Cashier extends Chef {
    public void calculateBill() {
        System.out.println("계산을 시작합니다");
        
        // 문제: 캐셔에게는 전혀 필요 없는 상수들이 자동으로 상속됨
        // KNIFE, PAN, SPATULA, OVEN, MICROWAVE 등이 모두 사용 가능
        // 하지만 캐셔는 요리를 하지 않으므로 의미 없음
    }
}

// 하위 클래스 2 (서빙 담당 - 요리와 무관)
public class Waiter extends Chef {
    public void serveFood() {
        System.out.println("음식을 서빙합니다");
        
        // 문제: 웨이터에게도 요리 도구 상수들이 자동으로 상속됨
        // KNIFE, PAN, SPATULA 등이 모두 사용 가능하지만 의미 없음
    }
}
```

**문제점:**
- 모든 하위 클래스가 필요하지 않은 상수들을 자동으로 상속받음
- 이름 충돌 가능성 (같은 이름의 상수를 정의하려 할 때)
- 하위 클래스 개발자가 혼란스러워함

---

## 3. 올바른 대안들

### 상수 유틸리티 클래스
```java
public class PhysicalConstants {
    private PhysicalConstants() { } // 인스턴스화 방지

    public static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    public static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    public static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

### 정적 임포트
```java
import static com.example.PhysicalConstants.*;

public class Calculator {
    public double calculateEnergy() {
        return AVOGADROS_NUMBER * BOLTZMANN_CONSTANT; // 클래스명 생략 가능
    }
}
```
### 열거 타입 

---

## 질문

Q1. 상수 인터페이스 안티패턴의 문제점은?

Q2. 상수 인터페이스의 대안은 무엇인가? 