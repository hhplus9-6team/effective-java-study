# ITEM18. 상속보다는 컴포지션을 사용하라

## 상속이란
- 자식 클래스는 부모 클래스의 자원을 물려 받아 재사용하는 것.
- 객체 간 관계를 `is-a 관계`로 표현한다.
  - Dog - Animal, Student - Person
- 적절한 사용 조건
  1) 두 클래스가 명확한 is-a 관계일 때만 사용한다.
  2) 상위 클래스가 확장을 고려해 설계/문서화되어 있을 때만 사용한다.

## 합성이란
- 기존 클래스를 확장하지 않고 필드로 해당 클래스의 인스턴스를 참조하여 기능을 재사용하는 것
- 객체 간 관계를 `has-a 관계` 로 표현한다.
  - Student - Subject, Car - Engine
  ```
  public class Car {
      private Engine engine;  // has-a 관계
  
      public Car(Engine engine) {
          this.engine = engine;
      }
  
      public void drive() {
          engine.start(); //포워딩 메서드
          System.out.println("자동차가 주행합니다.");
      }
  }
  ```
- `포워딩` : 새 클래스의 메서드가 내부의 기존 객체의 메서드를 호출하는 행위
- `포워딩 메서드` : 기조 객체의 메서드를 호출해 결과를 반환하는 메서드

## 상속과 합성 요약
| 구분     | 상속(Inheritance) | 합성(Composition)   |
| ------ | --------------- | ----------------- |
| 관계     | is-a 관계         | has-a 관계          |
| 결합 시점  | 컴파일 타임          | 런타임               |
| 결합도    | 높음 (구현 의존)      | 낮음 (인터페이스 의존)     |
| 관계 성격  | 정적 관계           | 동적 관계             |
| 재사용 방식 | 부모 클래스의 코드 상속   | 포함된 객체의 인터페이스 재사용 |
| 캡슐화    | 깨짐              | 유지됨               |
| 패턴     | 템플릿 메서드         | 데코레이터(Decorator)  |

## 상속의 단점
#### 1. 결합도가 높다.
- 부모-자식 관계가 컴파일 시점에 결정되므로 변경 유연성이 떨어진다.

#### 2. 불필요한 기능 상속
- 부모 클래스에 추가된 기능이 자식 클래스에는 부적절할 수 있다.
```
class Animal { void fly() {} }
class Tiger extends Animal {} //호랑이는 못 날지만 fly()를 물려받음
```

#### 3. 부모 클래스의 결함이 그대로 넘어온다.
- 상위 클래스의 버그나 실계 실수가 자식 클래스에도 그대로 반영된다.

#### 4. 부모 클래스와 자식 클래스의 동시 수정 문제
- 부모 클래스 수정 시, 자식 클래스 코드도 수정이 불가피해진다.

#### 5. 메서드 오버라이딩 오동작(캡슐화 파괴)
- 부모 클래스의 public 메서드가 내부적으로 다른 메서드를 호출하면, 오버라이드된 자식 메서드가 예상치 않게 호출될 수 있다.
```
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() { return addCount; }
}

InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
s.addAll(List.of("틱", "탁탁", "펑"));
System.out.println(s.getAddCount()); // 기대: 3, 실제: 6
```
- `HasSet.addAll()` 이 내부적으로 `add()` 를 호출한다.
- 자식 클래스의 `add()` 가 중복 호출되어 이중 증가 발생
- 즉, 상위 클래스의 구현 세부사항에 의존했기 때문에 깨진다.

#### 6. 불필요한 인터페이스 상속 문제
- `java.util.Stack`이 `Vector`를 상속하면 Stack의 특성과 어울리지 않는 insertElementAt() 등을 상속받음.

#### 7. 클래스 폭발
- 부모와 자식 클래스의 구현이 강하게 결합된 상속 관계 상태에서, 다양한 조합이 필요한 상황이 오면 결국 유일한 해결 방법은 조합의 수 만큼 새로운 클래스를 추가해 상속하는 것 뿐이다.
- 클래스 폭발 문제는 새로운 기능을 추가할 때뿐만 아니라 기능을 수정할 때에도 동일하게 발생한다.

#### 8. 단일 상속 제약
- Java는 단일 상속만 허용하므로, 이미 다른 클래스를 상속 중이면 추가 확장이 어렵다.

## 합성을 사용해야 하는 이유
- 상속보다 유연하고 안전하다.
- 내부 구현을 숨기므로 캡슐화가 유지된다.
- 런타임에 다른 구현체로 교체 가능 → 유연한 구조 설계 가능.
- 단점: 코드의 흐름(위임 구조)을 파악하기 어려워 약간 복잡해질 수 있다.

### 합성으로 해결
#### ForwardingSet - 전달 클래스
```
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;
    public ForwardingSet(Set<E> s) { this.s = s; }

    @Override public int size() { return s.size(); }
    @Override public boolean add(E e) { return s.add(e); }
    @Override public boolean addAll(Collection<? extends E> c) { return s.addAll(c); }
    // ... 나머지 메서드도 s에 위임
}
```
#### InstrumentedSet - 계측 기능 추가
```
public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;

    public InstrumentedSet(Set<E> s) { super(s); }

    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() { return addCount; }
}
```
```
Set<String> s = new InstrumentedSet<>(new HashSet<>());
s.addAll(List.of("틱", "탁탁", "펑"));
System.out.println(s.getAddCount()); // 3 (정상 동작)
```

#### InstrumentedSet 은 래퍼 클래스이자 데코레이터 패턴을 사용한다.
- 래퍼 클래스 
  - 다른 객체를 감싸 기능을 덧씌운 클래스
- 데코레이터 패턴
  - 기존 객체에 새로운 책임(기능)을 동적으로 추가하는 설계 패턴