# item87. 커스텀 직렬화 형태를 고려해보라

## 핵심 정리

클래스가 `Serializable`을 구현하고 기본 직렬화 형태를 사용하면, 다음 릴리스 때 버리려 한 현재의 구현에 영원히 발이 묶이게 된다. 기본 직렬화 형태를 버릴 수 없게 된다. 실제로 `BigInteger` 같은 일부 자바 클래스가 이 문제에 시달리고 있다.

먼저 고민해보고 괜찮다고 판단될 때만 기본 직렬화 형태를 사용하라. 기본 직렬화 형태는 유연성, 성능, 정확성 측면에서 신중히 고민한 후 합당할 때만 사용해야 한다. 일반적으로 직접 설계하더라도 기본 직렬화 형태와 거의 같은 결과가 나올 경우에만 기본 형태를 써야 한다.

---

## 기본 직렬화 형태란?

기본 직렬화 형태는 객체가 포함한 데이터뿐만 아니라 그 객체를 시작으로 접근할 수 있는 모든 객체와 객체들의 연결된 정보까지 나타낸다.
그러나 이상적인 직렬화 형태라면 물리적인 모습과 독립된 논리적인 모습만을 표현해야 한다.


---

## 기본 직렬화 형태에 적합한 경우

객체의 물리적 표현과 논리적 내용이 같다면 기본 직렬화 형태를 선택해도 무방하다.


```java
public class Name implements Serializable {
    /**
     * 성. null이 아니어야 함.
     * @serial
     */
    private final String lastName;
    
    /**
     * 이름. null이 아니어야 함.
     * @serial
     */
    private final String firstName;
    
    /**
     * 중간이름. 중간이름이 없다면 null.
     * @serial
     */
    private final String middleName;
    
    // ... 나머지 코드는 생략
}
```

- 성명은 논리적으로 이름, 성, 중간이름이라는 3개의 문자열로 구성된다.
- 위 클래스의 인스턴스 필드들은 이 논리적 구성요소를 정확하게 반영했다.
- 따라서 기본 직렬화 형태가 적합하다.

### 주의사항

기본 직렬화 형태가 적합하다고 결정했더라도 불변식 보장과 보안을 위해 `readObject` 메서드를 제공해야 할 때가 많다.
앞의 `Name` 클래스의 경우에는 `readObject` 메서드가 `lastName`, `firstName` 필드가 null이 아님을 보장해야 한다.


### @serial 태그의 역할

`Name` 클래스의 모든 필드는 `private`임에도 문서화 주석이 달려있다. 이는 이 필드들이 결국 클래스의 직렬화 형태에 포함되는 공개 API에 속하며, 공개 API는 모두 문서화해야 하기 때문이다.

@serial 태그의 역할:
- `private` 필드의 설명을 API 문서에 포함하라고 자바독에 알려주는 역할을 한다.
- `@serial` 태그로 기술한 내용은 API 문서에서 직렬화 형태를 설명하는 특별한 페이지에 기록된다.

---

## 기본 직렬화 형태에 적합하지 않은 경우

```java
public final class StringList implements Serializable {
    private int size = 0;
    private Entry head = null;
    
    private static class Entry implements Serializable {
        String data;
        Entry next;
        Entry previous;
    }
    
    // ... 나머지 코드는 생략
}
```

- 논리적: 이 클래스는 일련의 문자열을 표현한다.
- 물리적: 문자열들을 이중 연결 리스트로 연결했다.

이 클래스에 기본 직렬화 형태를 사용하면 각 노드의 양방향 연결 정보를 포함해 모든 엔트리(Entry)를 철두철미하게 기록한다.
기본 직렬화 형태를 사용하여 각 노드에 연결된 노드들까지 모두 표현된다.

물리적 표현과 논리적 표현의 차이가 클 때 기본 직렬화 형태를 사용하면 4가지 문제가 생긴다

#### 1. 공개 API가 현재의 내부 표현 방식에 영구히 묶인다

위 예제에서 private 클래스인 `Entry`가 공개 API가 되어버린다.

다음 릴리스에서 내부 표현 방식을 바꾸더라도 `StringList` 클래스는 여전히 연결 리스트로 표현된 입력도 처리할 수 있어야 한다. 즉, 연결 리스트의 관련 코드를 절대 제거할 수 없다.

#### 2. 너무 많은 공간을 차지하면서, 속도가 느려질 수 있다

앞 예의 직렬화 형태는 직렬화 형태에 포함될 가치가 없는 연결 리스트의 모든 엔트리와 연결 정보까지 기록한다.

#### 3. 시간이 너무 많이 걸린다

그래프를 직접 순회할 수밖에 없으므로 시간이 걸린다.

#### 4. 스택 오버플로를 일으킬 수 있다

기본 직렬화 과정은 객체 그래프를 재귀 순회하는데, 이 작업은 중간 정도 크기의 객체 그래프에서도 자칫 스택 오버플로를 일으킬 수 있다.

---

## 합리적인 커스텀 직렬화 형태

### StringList의 합리적인 직렬화 형태

위 `StringList` 예제의 합리적인 직렬화 형태는 무엇일까? 

단순히 리스트가 포함한 문자열의 개수를 적은 다음, 그 뒤로 문자열들을 나열하는 수준이면 될 것이다.
물리적인 상세 표현은 배제하고 논리적인 구성만을 담으면 된다.


```java
public final class StringList implements Serializable {
    private transient int size = 0;
    private transient Entry head = null;
    
    // 이번에는 직렬화 하지 않는다.
    private static class Entry {
        String data;
        Entry next;
        Entry previous;
    }
    
    // 지정한 문자열을 이 리스트에 추가한다.
    public final void add(String s) { 
        // ... 
    }
    
    /**
     * StringList 인스턴스를 직렬화한다.
     * @serialData 이 리스트의 크기(포함된 문자열 개수)가 먼저 기록되고,
     * 모든 원소(각각 String)가 순서대로 기록된다.
     */
    private void writeObject(ObjectOutputStream s)
            throws IOException {
        s.defaultWriteObject();
        s.writeInt(size);
        
        // 모든 원소를 올바른 순서대로 기록한다.
        for (Entry e = head; e != null; e = e.next) {
            s.writeObject(e.data);
        }
    }
    
    private void readObject(ObjectInputStream s)
            throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        int numElements = s.readInt();
        
        //모든 원소를 읽어 이 리스트에 삽인된다.
        for (int i = 0; i < numElements; i++) {
            add((String) s.readObject());
        }
    }
    
    // ... 나머지 코드는 생략
}
```

---

## transient 한정자

해당 인스턴스 필드가 기본 직렬화 형태에 포함되지 않는다는 표시다.

### defaultWriteObject와 defaultReadObject 호출

`StringList`의 필드 모두가 `transient`라도 `writeObject`, `readObject`는 각각 먼저 `defaultWriteObject`, `defaultReadObject`를 호출한다.
클래스의 인스턴스 필드 모두가 `transient`면 `defaultWriteObject`, `defaultReadObject`를 호출하지 않아도 된다고 들었을지 모르지만, 직렬화 명세는 이 작업을 무조건 하라고 요구한다.

#### 이유
- 그래야 `transient`가 아닌 인스턴스 필드가 추가된 다음 릴리스에서도 상호 호환되기 때문이다.
- 신버전의 인스턴스를 직렬화한 후에 구버전으로 역직렬화하면 새로 추가된 필드는 무시될 것이다.
- 구버전 `readObject` 메서드에서 `defaultReadObject`를 호출하지 않는다면 역직렬화 과정에서 `StreamCorruptedException`이 발생한다.

### transient 한정자 사용 원칙

`defaultWriteObject` 메서드를 호출하면 `transient`로 선언하지 않은 모든 인스턴스 필드에는 모두 `transient` 한정자를 붙여야 한다. 
캐시된 해시 값처럼 다른 필드에서 유도되는 필드도 여기 해당한다. 해당 객체의 논리적 상태와 무관한 필드라고 확신할 때만 `transient` 한정자를 생략해야 한다.
그래서 커스텀 직렬화 형태를 사용한다면, 대부분의 (혹은 모든) 인스턴스 필드를 `transient`로 선언해야 한다.

### transient 필드의 초기화

기본 직렬화를 사용한다면 `transient` 필드들은 역직렬화될 때 기본값으로 초기화된다.

만약 기본값을 그대로 사용하면 안 된다면:
- `readObject` 메서드에서 `defaultReadObject`를 호출한 다음, 해당 필드를 원하는 값으로 복원해야 한다.
- 혹은 그 값을 처음 사용할 때 초기화하는 방법도 있다.

---

## 동기화 메커니즘

### 원칙

객체의 전체 상태를 읽는 메서드에 적용해야 하는 동기화 메커니즘을 직렬화에도 적용해야 한다.

### 예시

모든 메서드를 `synchronized`로 선언하여 스레드 안전하게 만든 객체에서 기본 직렬화를 사용하려면 `writeObject`도 다음 코드처럼 `synchronized`로 선언해야 한다.

```java
private synchronized void writeObject(ObjectOutputStream stream)
        throws IOException {
    stream.defaultWriteObject();
}
```

### 주의사항

`writeObject` 메서드 안에서 동기화하고 싶다면 클래스의 다른 부분에서 사용하는 락 순서를 똑같이 따라야 한다. 그렇지 않으면 자원 순서 교착상태(resource-ordering deadlock)에 빠질 수 있다.

---

## 직렬 버전 UID를 명시적으로 부여

### 원칙

어떤 직렬화 형태를 택하든 직렬화 가능 클래스 모두에 직렬 버전 UID를 명시적으로 부여하자.

### 장점

1. 직렬 버전 UID가 일으키는 잠재적인 호환성 문제가 사라진다.
2. 직렬 버전 UID를 명시하지 않으면 런타임에 이 값을 생성하느라 복잡한 연산을 수행하는데, 이 수행 시간도 절약할 수 있다.

### 선언 방법

```java
private static final long serialVersionUID = <무작위로 고른 long 값>;
```

### 기존 클래스 수정 시

직렬 버전 UID가 없는 기존 클래스를 구버전으로 직렬화된 인스턴스와 호환성을 유지한 채 수정하고 싶다면, 구 버전에서 사용한 자동 생성된 값을 그대로 사용해야 한다. 

이 값은 직렬화된 인스턴스가 존재하는 구버전 클래스를 `serialver` 유틸리티에 입력으로 주어 실행하면 얻을 수 있다.

### 주의사항

구버전으로 직렬화된 인스턴스들과의 호환성을 끊으려는 경우를 제외하고는 직렬 버전 UID를 절대 수정하지 말자

---

## 결론

- 클래스를 직렬화하기로 했다면 어떤 직렬화 형태를 사용할지 심사숙고하기 바란다. 
- 자바의 기본 직렬화 형태는 객체를 직렬화한 결과가 해당 객체의 논리적 표현에 부합할 때만 사용하고, 
- 그렇지 않으면 객체를 적절히 설명하는 커스텀 직렬화 형태를 고안하라. 
- 직렬화 형태도 공개 메서드를 설계할 때에 준하는 시간을 들여 설계해야 한다.
- 한번 공개된 메서드는 향후 릴리스에서 제거할 수 없듯이
- 직렬화 형태에 포함된 필드도 마음대로 제거할 수 없다.
- 직렬화 호환성을 유지하기 위해 영원히 지원해야 하는 것이다. 
- 잘못된 직렬화 형태를 선택하면 해당 클래스의 복잡성과 성능에 영구히 부정적인 영향을 남긴다.

---
## QnA

Q1. 기본 직렬화 형태를 사용하면 왜 문제가 발생하나요?


---
## 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 87

