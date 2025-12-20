# item78. 공유 중인 가변 데이터는 동기화해 사용하라

> [!IMPORTANT]  
> 여러 스레드가 **공유하는 가변(mutable) 데이터**를 읽거나 쓴다면 **반드시 동기화**로 **원자성, 가시성**을 보장해야 한다.
>
> 핵심 포인트: 동기화는 배타적 실행만이 아니라 **스레드 간 안정적인 통신(가시성)** 을 위해서도 필요하다.

---

## synchronized(동기화)란?

`synchronized`는 메서드/블록을 **한 번에 한 스레드만** 수행하도록 보장한다.

- **배타적 실행(상태 일관성 보호)**: 같은 락(lock)으로 보호된 구역에는 동시에 하나의 스레드만 들어갈 수 있어,
  객체가 **일관되지 않은 중간 상태**로 관찰되는 일을 막는다.
- **스레드 간 통신(가시성)**: 락을 **획득/해제**하는 경계에서 메모리 가시성이 보장되어,
  한 스레드가 변경한 값이 다른 스레드에 **언제/어떻게 보이는지**가 정해진다.

즉, `synchronized`는 “한 명씩 실행” + “바뀐 내용이 서로 보이게 함”을 동시에 해결한다.

---

## 흔한 오해: “원자적으로 읽고/쓰면 동기화가 필요 없다?”

책에서는 **원자성**과 **가시성**을 분리해서 생각하라고 강조한다.

- Java 언어 명세상 `long`/`double`을 제외한 기본 타입의 읽기/쓰기는 원자적이다.
  - 반대로 말하면 `long`/`double`은(특히 `volatile`이 아니면) **찢어진 값** 을 볼 가능성이 있다.
- 하지만 **원자적으로 읽고/쓴다 = 최신 값을 본다**는 뜻이 아니다.
  - 동기화가 없으면 한 스레드의 변경이 다른 스레드에 **언제 보일지 보장되지 않는다(가시성 문제)**.

---

## 대표 예시 1) StopThread: 프로그램이 “영원히” 안 끝날 수 있다 (liveness failure)

StopThread 예시는 “메인 스레드가 1초 뒤 플래그를 true로 바꿨는데도 백그라운드가 종료되지 않을 수 있다”를 보여준다.

### ❌ 잘못된 코드(동기화 없음)
```java
public class StopThread {
    private static boolean stopRequested;

    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested) {
                i++;
            }
        });

        backgroundThread.start();
        java.util.concurrent.TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
```

### 무슨 일이 벌어지나? (가시성 + JIT 최적화 관점)

이 코드는 `stopRequested`를 여러 스레드가 공유하지만, 그 접근이 동기화(`synchronized`)나 `volatile` 없이 이뤄져서
자바 메모리 모델(JMM)에서 **"보이는 값(visibility)"** 이 보장되지 않기 때문에 끝나지 않을 수 있다.

- main 스레드가 `stopRequested = true;`로 값을 바꿔도, backgroundThread는 그 변경을 **영원히 못 볼 수 있다.**

이유는 크게 2가지가 겹친다.

#### 1) 가시성(visibility) 문제: 각 스레드는 “자기 메모리처럼” 볼 수 있음

JVM/CPU는 성능을 위해 값을 레지스터/코어 캐시에 올려두고 재사용할 수 있다.
동기화(`synchronized`)나 `volatile` 같은 **가시성 보장 장치**가 없으면,
backgroundThread는 `stopRequested`를 메인 메모리에서 다시 읽지 않고 **캐시에 있는 옛 값(false)** 만 계속 사용할 수 있다.

즉, main이 `true`로 바꿨다는 사실이 다른 스레드에 **전파된다는 보장 자체가 없다.**

#### 2) 컴파일러/JIT 최적화: 루프 밖으로 “읽기”를 빼버릴 수 있음(hoisting)

동기화가 없으면 JVM은 “이 값이 다른 스레드에서 바뀔 수 있다”는 제약을 덜 받기 때문에,
개념적으로 다음과 같은 최적화를 할 수도 있다:

```java
boolean cached = stopRequested; // 한 번만 읽음(false)
while (!cached) {
    i++;
}
```

같은 현상을 더 직관적으로 풀어쓰면 아래처럼 이해할 수도 있다(의미상 동일):

```java
if (!stopRequested) {
    while (true) {
        i++;
    }
}
```

- 결과적으로 `stopRequested`를 다시 읽지 않는 루프가 되어, 값이 바뀌어도 끝나지 않을 수 있다.

### 해결 1) 읽기/쓰기 모두를 `synchronized`로 보호
```java
public class StopThread {
    private static boolean stopRequested;

    private static synchronized void requestStop() { // 쓰기 메서드
        stopRequested = true;
    }

    private static synchronized boolean stopRequested() { // 읽기 메서드
        return stopRequested;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested()) {
                i++;
            }
        });

        backgroundThread.start();
        java.util.concurrent.TimeUnit.SECONDS.sleep(1);
        requestStop();
    }
}
```
- 쓰기만 동기화해서는 충분하지 않다.
- 읽기와 쓰기가 **같은 락 규칙** 아래 있어야 동작이 보장된다.

### 해결 2) “상태 공개(플래그)”는 `volatile`로도 가능
```java
private static volatile boolean stopRequested;
```
- `volatile`은 배타적 실행은 제공하지 않지만, **항상 가장 최근에 기록된 값을 읽게** 하는 “통신” 목적에는 적합하다.

---

## 대표 예시 2) volatile의 함정: `nextSerialNumber++`는 안전하지 않다 (safety failure)

`volatile`을 “조심해서 쓰라”고 하면서, `volatile`이어도 `++` 같은 **복합 연산(read-modify-write)** 은 원자성이 깨진다는 예시

### ❌ 잘못된 코드
```java
private static volatile int nextSerialNumber = 0;

public static int generateSerialNumber() {
    return nextSerialNumber++;
}
```

- `++`는 코드상 한 줄이지만 실제로는
  1) 값 읽기 → 2) 1 증가 → 3) 저장
  이라서 두 스레드가 끼어들면 **같은 값을 두 번 반환**할 수 있다.

### 해결 1) `synchronized`로 원자성 + 가시성 확보
```java
private static int nextSerialNumber = 0;

public static synchronized int generateSerialNumber() {
    return nextSerialNumber++;
}
```
- 이 경우 `volatile`은 불필요하므로 제거한다.

### 해결 2) `AtomicLong`(락-프리 동기화)
```java
private static final java.util.concurrent.atomic.AtomicLong nextSerialNum =
        new java.util.concurrent.atomic.AtomicLong();

public static long generateSerialNumber() {
    return nextSerialNum.getAndIncrement();
}
```


## 결론: 가장 좋은 방법은 “공유하지 않는 것”

- 가변 데이터를 **공유하지 말자**
- 가능하면 **불변 객체(아이템 17)** 만 공유하자
- 가변 객체를 다른 스레드에 넘겨야 한다면, **안전한 발행(safe publication)** 으로 공개하자

안전한 발행 방법
- 클래스 초기화 과정에서 정적 필드에 저장
- `volatile` 필드 / `final` 필드
- 락으로 보호되는 필드
- 동시성 컬렉션에 저장(아이템 81)


## 정리

- 동기화는 **배타적 실행(일관성)** 과 **스레드 간 통신(가시성)** 을 함께 제공한다.
- 원자적으로 읽고/쓴다고 해서 동기화가 불필요한 게 아니다(가시성은 별개).
- `volatile`은 플래그 같은 “통신 목적”엔 좋지만, `++` 같은 복합 연산의 원자성은 보장하지 못한다.
- 최선은 **공유 가변 상태를 없애고**, 불가피하면 `synchronized`/`Atomic*`/동시성 컬렉션으로 안전하게 다룬다.


## 질문(QnA)
Q1. `synchronized`가 제공하는 2가지 역할(배타적 실행 / 통신)을 설명하시오  
Q2. StopThread에서 읽는 쪽도 동기화가 필요한 이유는?
