# item79. 과도한 동기화는 피하라

멀티스레드 환경에서 동기화는 필수입니다.

하지만 필요 이상으로 동기화를 사용하면, 프로그램은 느려지거나 심지어 아무 오류 없이 멈춰버릴 수도 있습니다.

따라서, Item79 에서는 

> “동기화는 최소한으로, 그리고 그 안에서는 외부 코드를 호출하지 말라” 

라는 매우 중요한 원칙을 제시하고 있습니다.


## 1. 동기화의 목적

동기화(synchronized)의 목적은 오직 하나입니다.

> 공유 가변 상태(shared mutable state)를 보호하는 것

- 동시 접근으로 인한 데이터 깨짐 방지

- 가시성(visibility) 보장

이 목적을 넘어서 로직 실행, 콜백 호출, 작업 처리까지 동기화 안에 넣기 시작하면 문제가 생깁니다.

## 2. 과도한 동기화의 문제점

### 1) 성능 문제

- 락 경쟁(lock contention)

- 컨텍스트 스위칭 증가

- 스케일 아웃 불가 → 스레드를 늘려도 동시에 일하지 못하고, 처리량이 증가하지 않는 상태

### 2) 라이브니스 실패(liveness failure)

- 프로그램이 끝나지 않음

- 스레드가 영원히 대기

### 3) 교착 상태(deadlock)

- 여러 락이 얽혀 서로 영원히 기다림

## 3. 외계인 메서드(alien method)

외계인 메서드란 

> 내가 통제할 수 없는 코드

를 의미합니다.

예를 들면:

- 인터페이스 구현 메서드

- 콜백(listener)

- 오버라이드 가능한 메서드

- 사용자가 넘겨준 람다

- 외부 라이브러리 메서드


## 4. 왜 동기화 블록 안에서 외계인 메서드가 위험한가?

외계인 메서드가 내부에서:

- 오래 걸릴 수도 있고

- 다른 락을 잡을 수도 있고

- 다시 현재 객체의 락을 요청할 수도 있기 때문

이 상태에서 락을 쥔 채 호출하면:

- 다른 스레드는 전부 대기

- 교착 상태 가능

- 프로그램이 멈출 수 있음

## 5. 대표 예시: Observer 패턴

### 위험한 코드 예시

```java
synchronized void notifyObservers() {
    for (Observer o : observers) {
        o.update(this); // 외계인 메서드
    }
}
```

`update()`는 외부에서 구현된 것으로, 내부 동작을 전혀 보장할 수 없습니다.

### 발생 가능한 문제들

#### (1) 성능 문제

```
o.update() 안에서 Thread.sleep(1초)
```
-  락을 잡은 상태로 1초 정지
-  모든 스레드가 대기

#### (2) 교착 상태

```
update() 안에서 다른 락을 잡고,
그 락을 잡은 다른 스레드가 다시 이 객체 접근
```

- 서로 기다리며 멈춤

#### (3) ConcurrentModificationException

- ConcurrentModificationException 이 발생 가능
- ConcurrentModificationException은 멀티스레드 환경이 아니어도 발생할 수 있고, “Concurrent”라는 이름은 개념적 동시성(순회 중 수정)을 의미
- ConcurrentModificationException은 “동시에 수정돼서 위험하다”는 신호이지,
  스레드 안전을 보장해주는 장치가 아님

- 내부 동작 원리 (fail-fast)
  - 대부분의 java.util 컬렉션은 내부에 modCount라는 값을 가진다. 
  - 구조가 바뀔 때마다(add, remove) modCount++ 
  - Iterator는 생성 시점의 expectedModCount를 기억 
  - 순회 중에 두 값이 다르면 → 예외 발생
  ```java
    Iterator<E> it = list.iterator();
    while (it.hasNext()) {
        list.add(x); // modCount 변경
        it.next();   // 💥 ConcurrentModificationException
    }
  ```

- Observer 예제
    ```java
    for (Observer o : observers) {
        o.update(this);
    }
    ```
  만약 update() 안에서:
    ```java
    observers.remove(this);
    ```
  를 호출하면? 
  - observers 구조 변경 
  - 하지만 우리는 아직 for-each 순회 중 
  - 다음 순회 시점에 modCount != expectedModCount 
  - 💥 ConcurrentModificationException

## 6. 권장하는 해결 방법

```java
public void notifyObservers() {
    List<Observer> snapshot;
    synchronized (this) {
        snapshot = new ArrayList<>(observers);
    }
    for (Observer o : snapshot) {
        o.update(this); // 락 밖
    }
}
```

- 스냅샷(copy) + 락 분리
  - 동기화 영역 최소화
  - 외계인 메서드 락 밖에서 호출


## 7. 과도한 동기화를 피하는 실제적인 방법들

이 예시들은 모두 “과도한 동기화를 피하는 방법은 락을 더 잘 쓰는 것이 아니라, 아예 줄이거나 없애는 것”임을 보여줍니다.
자바는 기본 구현에서는 빠르고 단순하게 두고, 동시성이 필요할 때는 전용 도구(java.util.concurrent, ThreadLocalRandom 등)를 사용하도록 설계했습니다.

### 1) java.util vs java.util.concurrent

#### java.util 컬렉션 → 동기화 책임을 사용자에게 넘긴 예

- 기본적으로 스레드 안전하지 않음
- 동기화는 사용자 책임
- Iterator는 fail-fast (ConcurrentModificationException 발생)

#### java.util.concurrent 컬렉션 → 동시성 문제를 구조적으로 해결한 예

- 동시성을 설계에 포함
- 내부적으로 락 분할, CAS, 스냅샷 등 사용
- 외부 동기화 없이도 안전
- 대부분 CME 안 터짐 (또는 의미 없음)

```java
CopyOnWriteArrayList<Integer> list = new CopyOnWriteArrayList<>();

for (Integer i : list) {
    list.add(3); // 안전
}
```
- “순회 중 수정” 자체를 허용하는 설계

### 2) StringBuilder vs StringBuffer

#### StringBuffer

```java
StringBuffer sb = new StringBuffer();
sb.append("a"); // 항상 락
```

- 모든 메서드가 synchronized
- 스레드 안전
- 단일 스레드에서도 느림

#### StringBuilder

```java
StringBuilder sb = new StringBuilder();
sb.append("a");
```

- 동기화 없음
- 빠름
- 스레드 안전은 사용자가 책임

### 3) Random vs ThreadLocalRandom

#### Random

```java
Random r = new Random();
r.nextInt(); // synchronized
```

- 내부적으로 동기화
- 여러 스레드가 공유하면 병목

#### ThreadLocalRandom

```java
ThreadLocalRandom.current().nextInt();
```

- 스레드별 상태 유지
- 동기화 없음
- 스케일 아웃 가능

## 8. 결론

동기화는 반드시 필요하지만, 가장 위험한 동시성 도구이기도 합니다.

- 동기화는 공유 가변 상태 보호에만 사용하고
- 그 범위는 가능한 한 좁게 유지하며
- 동기화 블록 안에서는 외부 코드를 호출하지 말아야 합니다.

과도한 동기화 대신, 동시성에 맞게 설계된 자료구조와 패턴을 선택하는 것이 더 안전하고 확장 가능한 해결책입니다.

---

### 질문

Q. 다음 중 동기화 설계에 대한 설명으로 가장 적절한 것은 무엇인가요?

A) 안전을 위해 synchronized 블록은 최대한 넓게 잡는 것이 좋다.

B) 동기화는 공유 가변 상태를 보호하는 최소 범위로 제한해야 한다.

C) synchronized를 사용하면 교착 상태는 자동으로 방지된다.

<br>

Q. 이 코드에서 발생할 수 있는 문제는 무엇이고, 어떻게 수정하는 것이 좋을까요?

```java
public synchronized void addListener(Listener l) {
    listeners.add(l);
    l.onRegister();   // 외계인 메서드
}
```


---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 79