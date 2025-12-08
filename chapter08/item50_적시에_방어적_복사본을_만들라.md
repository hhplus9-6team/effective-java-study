# item 50. 적시에 방어적 복사본을 만들라
> * 방어적 복사는 클래스의 **무결성, 안정성, 보안성**을 지키기 위한 핵심 기법이다.
> * 특히 **가변 객체(Date, Calendar, 배열 등)**를 필드로 갖는 경우 필수적이다.
> * **“클라이언트가 제공한 매개변수가 신뢰할 수 있다고 가정하지 말라.”**
> * **“내부 상태를 외부에 노출하지 말라.”**

## 1. 오늘 배운 핵심 개념

* **방어적 복사(Defensive Copy)** 란, **클래스의 불변식(invariant)을 보존하고 보안 취약점을 막기 위해 외부로부터 전달되거나 외부로 나가는 객체를 복사하는 기법**이다.
* 특히 **가변(mutable) 객체가 필드에 포함된 클래스**에서 중요하다.
* 메서드나 생성자에서 **외부에서 받은 객체를 그대로 저장하거나, 내부 객체를 그대로 외부에 반환하면 위험**하다.

---

## 2. 왜 방어적 복사가 필요한가?

* 클라이언트가 전달한 객체가 **가변 객체라면 내부 상태가 훼손될 수 있다.**
* 의도치 않은 외부 변경 때문에 **불변식이 깨지고 버그, 보안 취약점으로 이어질 수 있다.**
* 반대로 내부의 가변 객체를 **그대로 반환하면 외부에서 내부 상태를 조작할 수 있게 된다.**

---

## 3. 생성자에서의 방어적 복사

### 3.1 문제 예시

```java
public Period(Date start, Date end) {
    this.start = start;   // 위험
    this.end = end;       // 위험
}
```

* Date는 가변 객체이므로, 생성 후에도 클라이언트가 `start.setTime(...)` 등을 호출해 내부 상태를 변경 가능.

### 3.2 해결: 방어적 복사 사용

```java
public Period(Date start, Date end) {
    this.start = new Date(start.getTime());
    this.end = new Date(end.getTime());

    if (this.start.after(this.end)) {
        throw new IllegalArgumentException();
    }
}
```

* **받은 매개변수를 바로 검증하지 않고 먼저 복사 후 검증해야 한다.**
* 그렇지 않으면 멀티스레드 환경에서 race condition 위험 있음.

---

## 4. 접근자(getter)에서의 방어적 복사

### 4.1 문제 예시

```java
public Date getStart() {
    return start; // 내부 가변 객체 그대로 반환
}
```

### 4.2 해결

```java
public Date getStart() {
    return new Date(start.getTime());
}
```

---

## 5. 방어적 복사 시 주의할 점

* **clone() 사용은 피하라.**

    * Date처럼 잘못된 clone 구현이 있을 수 있음.
    * 생성자를 활용한 복사 방식이 더 안전함.
* **방어적 복사는 비용이 든다.**

    * 하지만 보안/안정성 측면에서는 대부분 그 비용을 감수할 가치가 있다.
* **요구조건에 따라 복사를 생략해도 되는 경우가 있다.**

    * 예: 불변 객체만 전달받는다 확신할 수 있는 경우
    * 하지만 외부 API라면 절대 이런 가정을 하지 말아야 한다.

---

## 6. 방어적 복사가 불필요한 경우

* 전달받는 객체가 **불변 객체**일 때 (예: `LocalDate`, `Instant`)
* 클래스 자체가 **불변 클래스로 설계되어 있으며**, 내부에서 가변 객체를 절대 사용하지 않는 경우
* 불변식이 꺠지더라도 그 영향이 오직 호출한 클라이언트로 국한 될 때(Wrapper 클래스 패턴)

### 6.1. 래퍼 클래스 패턴 예시
```java
public class InstrumentedList<E> implements List<E> {
    private final List<E> list;
    private int addCount = 0;

    public InstrumentedList(List<E> list) {
        this.list = list; // 방어적 복사 없음
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return list.add(e);
    }

    public int getAddCount() {
        return addCount;
    }
}
```
```java
Set<String> raw = new HashSet<>();
InstrumentedSet<String> wrapper = new InstrumentedSet<>(raw);

// wrapper 통해 삽입
wrapper.add("A");

// raw를 직접 수정 — 래퍼의 불변식 깨짐
raw.add("B");
```
* 클라이언트가 원본 리스트를 직접 변경하면 wrapper의 addCount 불변식이 깨진다.
* raw를 외부에서 수정한 그 코드(클라이언트)만 알고 있는 문제이며
* 다른 클라이언트나 시스템 전체에는 영향을 미치지 않는다.
* 따라서 이런 상황에서는 방어적 복사를 강제할 필요가 없다.
> 책에 나온 위의 Period 클래스는 final 클래스이고 VO 객체이다.
> 예시로 든 Wrapper클래스는 클라이언트만 사용하는 객체라서 중요도가 다르다.


## QnA
- 방어적 복사는 언제 사용하는 것을 권장하는가?