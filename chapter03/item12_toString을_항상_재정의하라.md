# ITEM12: toString을 항상 재정의하라

## 핵심 개념
**toString()을 재정의하면 객체의 정보를 명확하고 유용하게 출력할 수 있습니다. 디버깅과 로깅에 매우 유용합니다.**

---

## toString()이란?

```java
public class Person {
    private String name;
    private int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

// 사용 예시
Person person = new Person("김철수", 25);
System.out.println(person);
// 출력: Person@1b6d3586 (클래스명@해시코드)
```

### 문제점
- **의미 없는 정보**: 해시코드만 보여줌
- **디버깅 어려움**: 객체 내용을 알 수 없음

---

## toString() 재정의

```java
public class Person {
    private String name;
    private int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + "}";
    }
}

// 사용 예시
Person person = new Person("김철수", 25);
System.out.println(person);
// 출력: Person{name='김철수', age=25}

public class PhoneNumber {
    private final int areaCode;
    private final int prefix;
    private final int lineNum;
    
    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = areaCode;
        this.prefix = prefix;
        this.lineNum = lineNum;
    }
    
    @Override
    public String toString() {
        return String.format("%03d-%04d-%04d", areaCode, prefix, lineNum);
    }
}

// 사용 예시
PhoneNumber phone = new PhoneNumber(10, 1234, 5678);
System.out.println(phone);
// 출력: 010-1234-5678
```

---

## 재정의가 불필요한 경우

- **정적 유틸리티 클래스**: 인스턴스가 없는 클래스
- **열거 타입**: 이미 적절한 toString() 제공
- **상위 클래스에서 이미 적절히 재정의한 경우**

---

## 예상 질문

### toString()을 재정의하면 좋은점?


