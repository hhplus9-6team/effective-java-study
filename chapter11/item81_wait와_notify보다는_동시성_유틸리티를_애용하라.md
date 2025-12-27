# item 81. wait와 notify보다는 동시성 유틸리티를 애용하라.
> * wait과 notify는 사양됐다고 보자. 
> * java.util.concurrent 를 사용하자.
>   * 실행자 프레임워크
>   * **동시성 컬렉션**
>   * **동기화 장치**

## 1. 동시성 컬렉션이란?
* List, Queue, Map과 같은 표준 컬렉션 인터페이스에 동시성을 가미해 구현한 고성능 컬렉션
* 동시성 컬렉션에서 동시성을 무력화하기는 불가능하고,
  외부에서 락을 추가로 사용하면 오히려 속도가 느려진다.
* 동시성 컬렉션은

  * 내부적으로 Lock을 사용하지만
  * **사용자 코드에서는 Lock을 의식하지 않게 해준다**
* 목적은

  * “락을 잘 쓰는 법”이 아니라
  * **“락을 직접 쓰지 않게 만드는 것”**

---

## 2. 일반 컬렉션과의 차이

### 2.1 일반 컬렉션 (HashMap, ArrayList)

* 스레드 안전하지 않음
* 웹 서버처럼 요청마다 스레드가 할당되는 환경에서

    * 동시에 `get / put` 발생 시 데이터 손상 가능
* 반복 중 수정 시 `ConcurrentModificationException` 발생

```java
Map<String, User> cache = new HashMap<>();
```

(static 하게 둔다면) 멀티스레드 웹 환경에서는 **그냥 쓰면 안 되는 컬렉션**이다.

---

### 2.2 synchronized 컬렉션

```java
Map<String, User> cache =
    Collections.synchronizedMap(new HashMap<>());
```

* 모든 연산이 **하나의 락**에 의존
* 읽기/쓰기 구분 없음
* 반복 시에도 외부에서 `synchronized` 필요
* 요청이 많아질수록 병목 심각

결과적으로

* 안전하지만
* 성능과 사용성 모두 좋지 않다.

---

### 2.3 동시성 컬렉션 (Concurrent Collections)

```java
Map<String, User> cache = new ConcurrentHashMap<>();
```

* 동기화 전략이 **컬렉션 내부에 캡슐화**
* 부분 락 / CAS 기반
* 외부에서 `synchronized` 사용 불필요
* 높은 동시성 보장

---

## 3. ConcurrentHashMap

### 3.1 무엇이 다른가?

* `HashMap`

    * 동시 접근 시 구조 자체가 깨질 수 있음
* `synchronizedMap`

    * 전체 맵에 하나의 락
* `ConcurrentHashMap`

    * 내부적으로 영역 분리
    * 읽기는 대부분 락 없이 수행
    * 쓰기도 충돌 범위 최소화

핵심은

* “Map 전체를 잠그지 않는다”는 점이다.
> ConcurrentHashMap은 락은 어떻게 걸릴까?
> * 읽기는 Lock X
> * 쓰기는 "해당 키가 들어갈 버킷(bin)에만 Lock O"

---

### 3.2 반복자(iteration)의 차이

```java
for (String key : map.keySet()) {
    ...
}
```

* `ConcurrentHashMap`의 반복자는

    * `ConcurrentModificationException`을 던지지 않음
    * **약한 일관성(weakly consistent)** 제공
* 최신 상태를 100% 보장하지는 않지만

    * 깨지지도 않는다

웹 서버에서 반복 중 수정 가능하다는 점은 매우 중요하다.

---

### 3.3 복합 연산을 안전하게 제공

일반 Map에서 흔히 발생하는 문제:

```java
if (!map.containsKey(key)) {
    map.put(key, value); // race condition
}
```

`ConcurrentHashMap`은 이를 위한 메서드를 제공한다.

```java
map.putIfAbsent(key, value);

map.computeIfAbsent(key, k -> loadValue());
```

* 여러 요청이 동시에 들어와도
* **한 번만 계산**
* 나머지는 같은 결과 공유

---

### 3.4 웹 개발에서의 실제 사용 예

#### 로그인 사용자 캐시

```java
@Component
public class LoginUserCache {

    private final ConcurrentHashMap<String, User> cache = new ConcurrentHashMap<>();

    public User getOrLoad(String token) {
        return cache.computeIfAbsent(token, this::loadUser);
    }

    private User loadUser(String token) {
        return userRepository.findByToken(token);
    }
}
```

* 동일 토큰으로 동시에 여러 요청이 와도
* DB 조회는 한 번만 수행
* 나머지는 캐시된 결과 사용

---

### 3.5 언제 사용하는가?

* 요청 간 공유 상태가 필요할 때
* 캐시
* 통계 집계
* 중복 호출 방지
* 글로벌 상태 저장소

정리하면

* **멀티스레드 환경에서 Map을 공유한다면 기본 선택은 ConcurrentHashMap**이다.

---

## 4. BlockingQueue

### 4.1 무엇을 해결하는 컬렉션인가?
> [생산자-소비자문제 예시](https://blog.naver.com/khjung1654/223876382457)
* 생산자–소비자 문제
* 큐가 비면 소비자는 기다려야 하고
* 큐가 가득 차면 생산자는 기다려야 함

이 대기/신호 로직을

* `wait / notify` 없이
* 컬렉션 내부에서 해결해준다.

---

### 4.2 동작 방식

```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>();

queue.put(task);   // 가득 차면 자동 대기
Task task = queue.take(); // 비어 있으면 자동 대기
```

* 락
* 조건 검사
* 신호 깨우기
  모두 내부에서 처리된다.

---

### 4.3 웹 개발에서의 대표적인 사용 상황

#### 요청은 빨리 응답해야 하지만, 부가 작업은 오래 걸리는 경우

* 로그 저장
* 메일 발송
* 외부 API 호출
* 통계 집계

요청 스레드에서 직접 처리하면

* 응답 지연
* 스레드 고갈 위험

---

### 4.4 웹 요청 + BlockingQueue 예시

#### 큐 정의

```java
@Component
public class AsyncLogQueue {

    private final BlockingQueue<LogEvent> queue = new LinkedBlockingQueue<>();

    public void enqueue(LogEvent event) throws InterruptedException {
        queue.put(event);
    }

    public LogEvent take() throws InterruptedException {
        return queue.take();
    }
}
```

---

#### 요청 처리 코드

```java
@PostMapping("/order")
public ResponseEntity<Void> order(@RequestBody OrderRequest req) {
    logQueue.enqueue(new LogEvent(req));
    return ResponseEntity.ok().build();
}
```

* 요청 스레드는

    * 작업을 큐에 넣고
    * 바로 응답

---

#### 백그라운드 워커

```java
@Component
public class LogWorker {

    @PostConstruct
    public void start() {
        new Thread(() -> {
            while (true) {
                try {
                    LogEvent event = logQueue.take();
                    saveLog(event);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }).start();
    }
}
```

* 작업 스레드는

    * 큐가 비면 자동 대기
    * 작업이 오면 자동 실행

---

### 4.5 언제 사용하는가?

* 요청 처리와 작업 실행을 분리하고 싶을 때
* 비동기 처리
* 작업 큐
* 이벤트 파이프라인

정리하면

* **“조건 대기”가 보이면 BlockingQueue 를 떠올리면 된다.**

---
* QnA
  * ConcurrentMap과 HashMap의 차이가 무엇인가요?