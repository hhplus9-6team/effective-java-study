**Enum의 ordinal() 대신 인스턴스 필드 사용하기**

Enum에는 `ordinal()`이라는 메서드가 있는데, 이건 그냥 상수가 몇 번째 위치인지 알려주는 거다. 근데 이걸 실제 값처럼 쓰면 안 된다. 대신 필요한 값은 인스턴스 필드에 직접 저장해서 써라.

---

**왜 ordinal()을 쓰면 안 될까**

Enum을 쓰다 보면 각 상수마다 숫자 값이 필요할 때가 있다. 예를 들어 합주단 종류를 나타내는 Enum이 있는데, 각 합주단에 몇 명이 연주하는지 알아야 하는 상황이라고 해보자.

이때 `ordinal()` 메서드를 쓰면 될 것 같아 보인다. 이 메서드는 상수가 몇 번째 위치인지 알려주니까.

```java
public enum Ensemble {
    SOLO, DUET, TRIO, QUARTET, QUINTET;

    public int numberOfMusicians() {
        return ordinal() + 1;  // 0번째부터 시작하니까 +1
    }
}
```

겉으로는 잘 돌아가는 것처럼 보이지만, 문제가 엄청 많다.

**문제 1: 순서를 바꾸면 망한다**
상수 순서만 살짝 바꿔도 값이 다 꼬인다. SOLO랑 DUET 순서를 바꾸면? 솔로가 2명, 듀엣이 1명이 되는 말도 안 되는 상황이 생긴다.

**문제 2: 같은 값을 가진 상수를 못 만든다**
8명이 연주하는 게 옥텟(OCTET) 하나만 있는 게 아니라 복4중주(DOUBLE_QUARTET)도 8명인데, 이걸 추가할 방법이 없다.

**문제 3: 중간을 비울 수가 없다**
12명이 연주하는 3중4중주를 추가하고 싶으면? 11번째 자리를 채워야 하는데, 11명으로 연주하는 합주단 이름이 없다. 그럼 억지로 더미 상수를 만들어야 한다. 진짜 비효율적이다.

**해결 방법: 인스턴스 필드 사용**

그냥 필요한 값을 필드에 직접 넣어주면 된다.

```java
public enum Ensemble {
    SOLO(1),
    DUET(2),
    TRIO(3),
    QUARTET(4),
    OCTET(8),
    DOUBLE_QUARTET(8),  // 같은 값도 가능
    TRIPLE_QUARTET(12);  // 중간 건너뛰기도 가능

    private final int numberOfMusicians;

    Ensemble(int size) {
        this.numberOfMusicians = size;
    }

    public int numberOfMusicians() {
        return numberOfMusicians;
    }
}
```

이렇게 하면:
- 순서 바꿔도 문제없음
- 같은 값 여러 개 가능
- 중간 건너뛰어도 됨
- 코드 의도가 명확함

Enum API 문서에도 나와 있다. ordinal()은 EnumSet이나 EnumMap 같은 특수한 자료구조 만들 때나 쓰라고 만든 거지, 일반적인 용도로 쓰지 말라고. 그러니까 그냥 쓰지 마라.

---

**동작 원리: 어떻게 값이 나오는 걸까**

`solo.numberOfMusicians()` 하면 어떻게 값이 나오는지 단계별로 보자.

**1단계: Enum 상수가 만들어질 때**

`SOLO(1)`이라고 쓰면, 자바가 자동으로 `new Ensemble(1)`을 호출하는 것과 비슷하다고 보면 된다.

```java
SOLO(1) 선언
    ↓
Ensemble 생성자 호출: Ensemble(1)
    ↓
this.numberOfMusicians = 1  // 필드에 1이 저장됨
    ↓
SOLO 상수 완성
```

**2단계: 프로그램 시작 시 자동으로 일어나는 일**

```java
SOLO 상수 = new Ensemble(1);
  └─ numberOfMusicians 필드 = 1

DUET 상수 = new Ensemble(2);
  └─ numberOfMusicians 필드 = 2

TRIO 상수 = new Ensemble(3);
  └─ numberOfMusicians 필드 = 3
```

**3단계: 메서드 호출**

```java
Ensemble solo = Ensemble.SOLO;  // SOLO 상수를 가져옴
int count = solo.numberOfMusicians();  // SOLO의 numberOfMusicians 필드 값인 1을 리턴
```

**일반 클래스로 비유하면**

```java
public class Person {
    private final String name;
    private final int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public int getAge() {
        return age;
    }
}

// 사용할 때
Person kim = new Person("김철수", 20);
int age = kim.getAge();  // 20이 나옴
```

이거랑 똑같은 원리다. Enum도 내부적으로는 객체고, 생성자로 값을 받아서 필드에 저장한다.

**핵심 정리**

1. `SOLO(1)` 선언 → 생성자 `Ensemble(1)` 호출
2. 생성자에서 `this.numberOfMusicians = 1` 실행
3. SOLO 상수의 numberOfMusicians 필드에 1이 저장됨
4. 나중에 `solo.numberOfMusicians()` 호출하면 저장된 1이 나옴

Enum 상수 하나하나가 사실은 객체라서, 생성자로 값을 받아서 필드에 저장하고, 메서드로 그 값을 꺼내 쓰는 거다.

---

**실제 사용 예시**

**기본 사용법**

```java
public class Concert {
    public static void main(String[] args) {
        Ensemble solo = Ensemble.SOLO;
        Ensemble octet = Ensemble.OCTET;
        Ensemble doubleQuartet = Ensemble.DOUBLE_QUARTET;

        System.out.println(solo.name() + "는 " + solo.numberOfMusicians() + "명이 필요합니다.");
        System.out.println(octet.name() + "는 " + octet.numberOfMusicians() + "명이 필요합니다.");
        System.out.println(doubleQuartet.name() + "는 " + doubleQuartet.numberOfMusicians() + "명이 필요합니다.");

        // 출력:
        // SOLO는 1명이 필요합니다.
        // OCTET는 8명이 필요합니다.
        // DOUBLE_QUARTET는 8명이 필요합니다.
    }
}
```

**공연장 예약 시스템**

```java
public class ConcertHall {
    public static boolean canBook(Ensemble ensemble, int availableSeats) {
        int required = ensemble.numberOfMusicians();

        if (required <= availableSeats) {
            System.out.println(ensemble.name() + " 예약 가능 (필요: " + required + "명)");
            return true;
        } else {
            System.out.println(ensemble.name() + " 예약 불가 (필요: " + required + "명, 가능: " + availableSeats + "명)");
            return false;
        }
    }

    public static void main(String[] args) {
        int stage = 5;  // 무대에 5명만 설 수 있음

        canBook(Ensemble.SOLO, stage);      // 가능
        canBook(Ensemble.TRIO, stage);      // 가능
        canBook(Ensemble.OCTET, stage);     // 불가
    }
}
```

**연주자 급여 계산**

```java
public class PaymentCalculator {
    private static final int PAY_PER_MUSICIAN = 100000;  // 1인당 10만원

    public static int calculateTotalPay(Ensemble ensemble) {
        return ensemble.numberOfMusicians() * PAY_PER_MUSICIAN;
    }

    public static void main(String[] args) {
        for (Ensemble e : Ensemble.values()) {
            int total = calculateTotalPay(e);
            System.out.println(e.name() + ": " + total + "원");
        }

        // 출력:
        // SOLO: 100000원
        // DUET: 200000원
        // TRIO: 300000원
        // QUARTET: 400000원
        // OCTET: 800000원
        // DOUBLE_QUARTET: 800000원
        // TRIPLE_QUARTET: 1200000원
    }
}
```

**무대 크기 결정**

```java
public class StageManager {
    public static String getStageSize(Ensemble ensemble) {
        int musicians = ensemble.numberOfMusicians();

        if (musicians == 1) {
            return "소형 무대";
        } else if (musicians <= 4) {
            return "중형 무대";
        } else if (musicians <= 9) {
            return "대형 무대";
        } else {
            return "특대형 무대";
        }
    }

    public static void main(String[] args) {
        System.out.println(Ensemble.SOLO.name() + " -> " + getStageSize(Ensemble.SOLO));
        System.out.println(Ensemble.QUARTET.name() + " -> " + getStageSize(Ensemble.QUARTET));
        System.out.println(Ensemble.OCTET.name() + " -> " + getStageSize(Ensemble.OCTET));
        System.out.println(Ensemble.TRIPLE_QUARTET.name() + " -> " + getStageSize(Ensemble.TRIPLE_QUARTET));
    }
}
```
**1단계: 프로그램 시작**
```
StageManager.main() 메서드 실행 시작
```

**2단계: Ensemble.SOLO 첫 참조 → 클래스 로딩**
```
Ensemble.SOLO를 만나는 순간
    ↓
JVM: "아, Ensemble 클래스를 아직 로딩 안 했네?"
    ↓
Ensemble 클래스 로딩 시작
    ↓
모든 enum 상수 생성 (생성자 호출)
```
이렇게 보면 `numberOfMusicians()` 메서드가 실제로 얼마나 유용한지 알 수 있다. ordinal()을 썼으면 순서 바뀔 때마다 모든 로직이 망가졌을 거고, OCTET과 DOUBLE_QUARTET처럼 같은 값을 가진 상수도 못 만들었을 것이다.

--------

## 이해도 테스트문

**문제: 다음 중 Enum에서 ordinal() 메서드를 사용하지 말아야 하는 이유로 적절하지 않은 것은?**

```java
public enum PaymentType {
    CARD, CASH, TRANSFER, POINT;

    public int getCode() {
        return ordinal() + 1;
    }
}
```

1. 상수의 순서를 변경하면 getCode() 메서드가 반환하는 값도 함께 변경되어 예상치 못한 버그가 발생할 수 있다.

2. CARD와 CASH 둘 다 코드 값으로 1을 사용하고 싶어도 ordinal()로는 서로 다른 값을 가진 상수를 만들 수 없다.

3. 코드 값 5를 사용하고 싶어도 4번째 위치에 더미 상수를 추가해야 하므로 코드가 불필요하게 복잡해진다.

4. ordinal() 메서드는 private 접근 제어자로 선언되어 있어서 외부 클래스에서 호출할 수 없다.

---

**정답을 맞춰보고, 왜 그렇게 생각했는지 말해줘.**
