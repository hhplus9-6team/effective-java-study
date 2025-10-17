# item20_추상클래스보다 인터페이스를 우선하라

여러분이 **게임 캐릭터**를 만든다고 상상해보세요!

```java
// 😱 문제 상황: 추상 클래스만 쓸 때
abstract class 전사 {
    abstract void 칼휘두르기();
}

abstract class 마법사 {
    abstract void 마법쓰기();
}

// 🤯 마검사(마법도 쓰고 칼도 쓰는)를 만들고 싶은데...
// class 마검사 extends 전사, 마법사 { }  // ❌ 에러! 자바는 다중상속 안됨!
```

**"아... 마검사는 못 만드나?"** 😭

## ✨ 인터페이스가 해결해줍니다!

```java
// 😎 해결: 인터페이스 사용
interface 전사 {
    void 칼휘두르기();
}

interface 마법사 {
    void 마법쓰기();
}

interface 궁수 {
    void 활쏘기();
}

// 🎉 이제 뭐든 조합 가능!
class 마검사 implements 전사, 마법사 {
    public void 칼휘두르기() { 
        System.out.println("슥삭! ⚔️"); 
    }
    public void 마법쓰기() { 
        System.out.println("파이어볼! 🔥"); 
    }
}

// 심지어 3개도 가능!
class 만능캐릭터 implements 전사, 마법사, 궁수 {
    // 구현...
}
```

## 🍕 실생활 비유: 피자 토핑처럼!

```java
// 인터페이스 = 토핑 (여러 개 올릴 수 있음)
interface 페퍼로니 { }
interface 치즈추가 { }
interface 올리브 { }

class 나만의피자 implements 페퍼로니, 치즈추가, 올리브 {
    // 원하는 토핑 다 올리기! 🍕
}

// 추상클래스 = 도우 (하나만 선택)
abstract class 씬도우 { }
abstract class 두꺼운도우 { }
// class 피자 extends 씬도우, 두꺼운도우 { }  // ❌ 도우는 하나만!
```

## 📱 Java 8의 꿀기능: 디폴트 메서드

```java
interface 스마트폰 {
    void 전화하기();
    void 문자보내기();
    
    // 🎁 Java 8부터: 기본 구현 제공 가능! -> 구현방법이 명백하다면 개발자들의 일감을 덜어줌.
    default void 셀카찍기() {
        카메라켜기();
        System.out.println("찰칵! 📸");
    }
    
    private void 카메라켜기() {  // Java 9부터 private 메서드도 가능
        System.out.println("카메라 앱 실행");
    }
}

class 아이폰 implements 스마트폰 {
    public void 전화하기() { System.out.println("따르릉"); }
    public void 문자보내기() { System.out.println("iMessage 전송"); }
    // 셀카찍기()는 안 만들어도 이미 사용 가능!
}
```

## 🏗️ 골격 구현: 인터페이스 + 추상클래스 조합

**"인터페이스는 좋은데... 공통 코드를 매번 다시 짜기 귀찮아요"** 😫

```java
// 1️⃣ 인터페이스로 규격 정의
interface 자동차 {
    void 시동걸기();
    void 주행하기();
    void 정지하기();
}

// 2️⃣ 추상클래스로 공통 부분 구현 (Abstract~ 이름 규칙)
abstract class Abstract자동차 implements 자동차 {
    // 공통 로직은 여기서 구현
    public void 시동걸기() {
        System.out.println("키를 돌립니다");
        System.out.println("부르릉~ 🚗");
    }
    
    public void 정지하기() {
        System.out.println("브레이크 밟기!");
    }
    
    // 차종마다 다른 부분은 하위에서 구현
    public abstract void 주행하기();
}

// 3️⃣ 실제 구현은 간단!
class 스포츠카 extends Abstract자동차 {
    public void 주행하기() {
        System.out.println("시속 200km로 슝~! 🏎️");
    }
}

class 트럭 extends Abstract자동차 {
    public void 주행하기() {
        System.out.println("무거운 짐 싣고 천천히~ 🚚");
    }
}

// 실제 사용해보기
public class Main {
    public static void main(String[] args) {
        스포츠카 myCar = new 스포츠카();

        myCar.시동걸기();  // ✅ 바로 사용 가능!
        // 출력: 
        // 키를 돌립니다
        // 부르릉~ 🚗

        myCar.주행하기();  // ✅ 내가 구현한 메서드
        // 출력: 시속 200km로 슝~! 🏎️

        myCar.정지하기();  // ✅ 바로 사용 가능!
        // 출력: 브레이크 밟기!
    }
}
```



## 🎯 핵심 정리: 3줄 요약

1. **인터페이스 = 레고 블록** 🧱  
   여러 개를 조합해서 원하는 모양 만들기 가능!

2. **추상클래스 = 붕어빵 틀** 🥞  
   틀은 하나만 쓸 수 있지만, 속 재료는 바꿀 수 있음

3. **둘 다 쓰기 = 최고의 선택** 💪  
   인터페이스로 설계하고, 골격 구현(Abstract~)으로 편하게!

---

## ✅ 이해도 확인 문제

### 문제: 다음 중 인터페이스의 장점이 **아닌** 것은?

```java
// 상황: 온라인 쇼핑몰 결제 시스템 개발 중
```

**A)** 하나의 클래스가 카드결제, 포인트결제, 쿠폰할인을 모두 구현할 수 있다
```java
class 결제시스템 implements 카드결제, 포인트결제, 쿠폰할인 { }
```

**B)** 이미 만들어진 클래스에 나중에 새로운 기능을 추가하기 쉽다
```java
class 기존결제 { }  // 나중에 implements 환불가능 추가 가능
```

**C)** 인터페이스는 인스턴스 변수(필드)를 가질 수 있어서 데이터 저장이 편리하다
```java
interface 결제 {
    private int amount;  // 이게 가능할까?
}
```

**D)** Java 8부터 디폴트 메서드로 공통 로직을 제공할 수 있다
```java
interface 결제 {
    default void 영수증출력() { System.out.println("영수증 출력 중..."); }
}
```
