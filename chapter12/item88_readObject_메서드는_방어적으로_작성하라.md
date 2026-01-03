# item88. readObject 메서드는 방어적으로 작성하라

객체 직렬화란 자바가 객체를 바이트 스트림으로 인코딩하고(직렬화) 그 바이트 스트림으로부터 다시 객체를 재구성하는(역직렬화) 메커니즘이다.
직렬화된 객체는 다른 VM에 전송하거나 디스크에 저장한 후 나중에 역직렬화할 수 있다.
직렬화가 품고 있는 위험과 그 위험을 최소화하는 방법에 대해 알아보자.

Item50 적시에 방어적 복사본을 만들라에서는 불변인 날짜 범위 클래스를 만드는데 있어 가변인 Date필드를 이용했다. 
그래서 불변식을 지키고 불변을 유지하기 위해 생성자와 접근자에서 Date 객체를 방어적으로 복사하느라 코드가 길어졌다.
```
// 방어적 복사를 사용하는 불변 클래스
public final class Period {
    private final Date start;
    private final Date end;

    /**
     * @param  start 시작 시각
     * @param  end 종료 시각; 시작 시각보다 뒤여야 한다.
     * @throws IllegalArgumentException 시작 시각이 종료 시각보다 늦을 때 발생한다.
     * @throws NullPointerException start나 end가 null이면 발생한다.
     */
    public Period(Date start, Date end) {
        this.start = new Date(start.getTime()); // 가변인 Date 클래스의 위험을 막기 위해 새로운 객체로 방어적 복사를 한다.
        this.end = new Date(end.getTime());

        if (this.start.compareTo(this.end) > 0) {
            throw new IllegalArgumentException(start + " after " + end);
        }
    }

    public Date start() { return new Date(start.getTime()); }
    public Date end() { return new Date(end.getTime()); }
    public String toString() { return start + " - " + end; }
    // ... 나머지 코드는 생략
}
```

## 이 클래스를 직렬화하기로 가정한다면,
직렬화를 하기 위해서 과연 `implements Serializable` 만 추가하면 될까?
- Period 클래스는 물리적 표현과 논리적 표현이 같기 때문에 기본 직렬화 형태를 사용해도 나쁘지 않다.
  - 물리적 표현 : 객체가 메모리에 실제로 저장되는 구조
  - 논리적 표현 : 이 객체가 개념적으로 의미하는 값
  - 실제로 두개의 Date 가 저장되고 개념적으로 시작 시각과 종료 시각으로 이루어진 기간으로 내부 필드 구조가 외부에 노출되는 개념 모델과 1:1로 대응한다.
- 하지만 실제로는 불변식을 보장하지 못한다.
  - 생성자는 start <= end 불변식을 보장한다.
  - 하지만 기본 직렬화는 생성자를 호출하지 않고, 바이트 스트림에 있는 값을 리플렉션으로 필드에 그대로 주입하기 때문이다.
  - 그래서 유효성 검사, 방어적 복사, 불변식 검사가 이루어지지 않는다.
- readObject 가 또다른 public 생성자이기 때문에 인수가 유효한지 검사해야 하고, 필요하다면 매개변수를 방어적으로 복사까지 해야 한다.
  - readObject 가 이 작업을 제대로 수행하지 못하면 공격자는 손쉽게 해당 클래스의 불변식을 깨뜨릴 수 있다.
```
// 종료 시각이 시작 시간보다 앞서는 Peiod 인스턴스를 만들 수 있다. 
public class BogusPeriod {
    // 불변식을 깨뜨리도록 조작된 바이트 스트림
    private static final byte[] serializedForm = {
        (byte)0xac, (byte)0xed, 0x00, 0x05, 0x73, 0x72, 0x00, 0x06,
        0x50, 0x65, 0x72, 0x69, 0x6f, 0x64, 0x40, 0x7e, (byte)0xf8,
        0x2b, 0x4f, 0x46, (byte)0xc0, (byte)0xf4, 0x02, 0x00, 0x02,
        0x4c, 0x00, 0x03, 0x65, 0x6e, 0x64, 0x74, 0x00, 0x10, 0x4c,
        0x6a, 0x61, 0x76, 0x61, 0x2f, 0x75, 0x74, 0x69, 0x6c, 0x2f,
        0x44, 0x61, 0x74, 0x65, 0x3b, 0x4c, 0x00, 0x05, 0x73, 0x74,
        0x61, 0x72, 0x74, 0x71, 0x00, 0x7e, 0x00, 0x01, 0x78, 0x70,
        0x73, 0x72, 0x00, 0x0e, 0x6a, 0x61, 0x76, 0x61, 0x2e, 0x75,
        0x74, 0x69, 0x6c, 0x2e, 0x44, 0x61, 0x74, 0x65, 0x68, 0x6a,
        (byte)0x81, 0x01, 0x4b, 0x59, 0x74, 0x19, 0x03, 0x00, 0x00,
        0x78, 0x70, 0x77, 0x08, 0x00, 0x00, 0x00, 0x66, (byte)0xdf,
        0x6e, 0x1e, 0x00, 0x78, 0x73, 0x71, 0x00, 0x7e, 0x00, 0x03,
        0x77, 0x08, 0x00, 0x00, 0x00, (byte)0xd5, 0x17, 0x69, 0x22,
        0x00, 0x78
    };

    public static void main(String[] args) {
        Period p = (Period) deserialize(serializedForm);
        System.out.println(p.start);
        System.out.println(p.end);
        // Fri Jan 01 12:00:00 PST 1999 // start 가 더 느리다.
        // Sun Jan 01 12:00:00 PST 1984 // end 가 더 이르다.
    }

    static Object deserialize(byte[] sf) {
        try {
            return new ObjectInputStream(new ByteArrayInputStream(sf)).readObject();
        } catch (IOException | ClassNotFoundException e) {
            throw new IllegalArgumentException(e);
        }
    }
}
```

## 해결방법
- readObject 를 정의하고, 유효성 검사를 실시한다.
- Period 클래스에 다음의 메서드를 추가한다.
  - readObject 메서드가 defaultReadObject 를 호출한 다음 역직렬화된 객체가 유효한지 검사해야 한다.
  - 이 유효성 검사에 실패하면 `InvalidObjectException` 을 던지게 하여 잘못된 역직렬화가 일어나는 것을 막을 수 있다.
```
// 유효성 검사를 수행하는 readObject 메서드 - 아직 부족하다!
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
        s.defaultReadObject(); // 기본 직렬화 수행

        if (start.compareTo(end) > 0) { // 유효성 검사
            throw new InvalidObjectException(start+" 가 "+end+" 보다 늦을 수 없습니다.");
        }
}
```
- readObject 메서드가 방어적 복사를 충분히 하지 않은 데 있다. 
  - start , end 가 외부에서 변경되지 않았고,
  - start, end 를 Period 만 소유했다.
  - 즉, 지금 이 순간의 값만 정상이지 앞으로도 절대 변하지 않는다는 보장이 없다.
- 객체를 역직렬화할때는 클라이언트가 소유해서는 안되는 객체 참조를 갖는 필드를 모두 반드시 방어적으로 복사해야 한다.
  - defaultReadObject 의 의미 : 바이트 스트림 안에 있던 객체를 그대로 Period 내부 필드에 꽂아 넣는다.
  - Date 의 소유권이 외부에 있을 수 있다.
  - 방어적 복사는 소유권을 끊는 것이다.
- 따라서 readObjejct 에서는 불변 클래스 안의 모든 private 가변 요소를 방어적으로 복사해야 한다.
```
// 방어적 복사와 유효성 검사를 수행하는 readObject 메서드
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
        s.defaultReadObject();

        // 가변 요소들을 방어적으로 복사한다.
        start = new Date(start.getTime());
        end = new Date(end.getTime());

        // 불변식을 만족하는지 검사한다.
        if (start.compareto(end) > 0) {
            throw new InvalidObjectException(start + " after " + end);
        }
}
```

## 결론
readObject 메서드를 작성할 때는 언제나 public 생성자를 작성하는 자세로 임해야 한다.
readObject 는 어떤 바이트 스트림이 넘어오더라도 유효한 인스턴스를 만들어내야 한다.
바이트 스트림이 진짜 직렬화된 인스턴스라고 가정해서는 안된다.

### 안전한 readObject 메서드를 작성하는 지침
- private 이어야 하는 객체 참조 필드는 각 필드가 가리키는 객체를 방어적으로 복사하라
- 모든 불변식을 검사하여 어긋나는게 발견되면 `InvalidObjectException` 을 던진다.
- 역직렬화 후 객체 그래프 전체의 유효성을 검사해야 한다면 `ObjectInputValidation` 인터페이스를 사용해라
- 직접적이든 간접적이든 재정의할 수 있는 메서드를 호출하지 말자

## 질문
- 기본 직렬화가 아닌 readObject 메서드를 방어적으로 작성해야 하는 이유가 무엇인가요?