# item37. ordinal 인덱싱 대신 EnumMap을 사용하라

## 1. ordinal 인덱싱

```java
enum Phase { SOLID, LIQUID, GAS }

enum Transition {
    MELT, FREEZE, BOIL, CONDENSE, SUBLIME, DEPOSIT;

    // 상태 전이 정의 (배열 인덱스 사용)
    private static final Transition[][] TRANSITIONS = {
        { null, MELT, SUBLIME },
        { FREEZE, null, BOIL },
        { DEPOSIT, CONDENSE, null }
    };

    public static Transition from(Phase from, Phase to) {
        return TRANSITIONS[from.ordinal()][to.ordinal()];
    }
}
```

### 1) ordinal 인덱싱이란?

- ordinal 인덱싱이란 자바 열거 타입의 `ordinal()` 메서드를 이용해서 열거 상수의 선언 순서를 배열 인덱스로 사용하는 방식
- `ordinal()` 메서드는 상수가 선언된 순서를 0부터 반환
- `from.ordinal()`과 `to.ordinal()`을 통해 이차원 배열 접근

### 2) 문제점

- 열거형 순서가 바뀌거나 새로운 상수가 추가되면, 인덱스 기반 배열이 전부 틀어질 수 있습니다.
- `ordinal()` 은 코드만 봐서는 의미가 불명확하여 가독성을 저하시킵니다.
- 상수 추가 시 배열 크기와 인덱싱 로직을 모두 수정해야 해서 유지보수가 어렵습니다.
- `ordinal()` 은 단순히 숫자일 뿐이므로, 잘못된 enum 타입을 써도 컴파일러가 잡지 못하여 타입 안전성이 없습니다.

## 2. EnumMap

```java
enum Phase {
    SOLID, LIQUID, GAS;

    enum Transition {
        MELT(SOLID, LIQUID),
        FREEZE(LIQUID, SOLID),
        BOIL(LIQUID, GAS),
        CONDENSE(GAS, LIQUID),
        SUBLIME(SOLID, GAS),
        DEPOSIT(GAS, SOLID);

        private final Phase from;
        private final Phase to;

        Transition(Phase from, Phase to) {
            this.from = from;
            this.to = to;
        }

        // EnumMap 기반 전이표
        private static final Map<Phase, Map<Phase, Transition>> m = new EnumMap<>(Phase.class);

        static {
            for (Phase p : Phase.values())
                m.put(p, new EnumMap<>(Phase.class));
            for (Transition trans : values())
                m.get(trans.from).put(trans.to, trans);
        }

        public static Transition from(Phase from, Phase to) {
            return m.get(from).get(to);
        }
    }
}
```
### 1) EnumMap 이란?

- 열거 타입을 키로 사용하는 고성능 Map 구현체
- 내부적으로 배열을 이용해 ordinal 기반으로 동작하지만, ordinal 인덱싱의 복잡함과 위험을 캡슐화함
- 배열만큼 빠르지만 타입 안전성과 가독성 확보

### 2) 장점

- EnumMap 은 열거 타입을 키로 사용하므로 타입 안전성이 보장됩니다.
- ordinal 인덱싱은 숫자 기반 접근이라 의미가 불명확하지만, EnumMap은 이름 그대로 사용해 가독성이 높습니다.
- ordinal 기반 배열은 상수 순서가 바뀌면 오류가 발생하지만, EnumMap은 순서 변경과 무관해 유지보수가 쉽습니다.
- 내부적으로 배열을 사용하기 때문에 EnumMap의 성능은 ordinal 인덱싱과 거의 동일합니다.

### 3) EnumMap 의 내부 구조

```java
public class EnumMap<K extends Enum<K>, V> extends AbstractMap<K,V> {
    private final Class<K> keyType;   // 키의 타입 저장
    private final K[] keyUniverse;    // 모든 enum 상수 배열
    private final Object[] vals;      // 값 저장용 배열 (ordinal 인덱스 기반)
}
```

- EnumMap은 내부적으로 `Object[]` 배열을 하나 가지고 있습니다.
- 이 배열의 인덱스는 `key.ordinal()` 을 기반으로 접근하지만, 사용자는 오직 enum 상수 키로만 접근하게 되어 타입 안전성이 유지됩니다.
- 내부적으로는 ordinal 을 사용하지만 외부에는 안전하게 감춰서, 배열 기반처럼 빠르지만 안전하게 타입 체클를 수행합니다.

### 4) EnumMap 중첩

```java
private static final Map<Phase, Map<Phase, Transition>> m = new EnumMap<>(Phase.class);
```

- 상태 전이 같은 2차원 관계를 표현할 때는 `Map<K, Map<V, R>>` 형태로 중첩 가능합니다.
- 내부 맵도 EnumMap으로 구성하면 동이하게 타입 안전하고 효율적입니다.

### 5) 스트림 활용

```java
enum Phase {
    SOLID, LIQUID, GAS;

    enum Transition {
        MELT(SOLID, LIQUID),
        FREEZE(LIQUID, SOLID),
        BOIL(LIQUID, GAS),
        CONDENSE(GAS, LIQUID),
        SUBLIME(SOLID, GAS),
        DEPOSIT(GAS, SOLID);

        private final Phase from;
        private final Phase to;

        Transition(Phase from, Phase to) {
            this.from = from;
            this.to = to;
        }

        private static final Map<Phase, Map<Phase, Transition>> m =
                Arrays.stream(values())
                      .collect(Collectors.groupingBy(
                              t -> t.from,
                              () -> new EnumMap<>(Phase.class), // 외부 맵
                              Collectors.toMap(
                                      t -> t.to,
                                      t -> t,
                                      (x, y) -> y,
                                      () -> new EnumMap<>(Phase.class) // 내부 맵
                              )));

        public static Transition from(Phase from, Phase to) {
            return m.get(from).get(to);
        }
    }
}
```

- 반복문 대신 스트림을 사용하면 초기화 코드를 더 짧고 선언적으로 작성할 수 있습니다.
- 로직이 명시적으로 드러나므로 가독성과 유지보수성이 높습니다.
- 디버깅 시 Map 구조를 추적하기 어렵고 IDE 인식이 떨어질 수 있습니다.
  - 중간 단계(예: groupingBy가 어떤 Map을 만들고 있는지, 각 Phase별로 어떤 Transition이 쌓였는지)를 디버거에서 쉽게 볼 수 없습니다.
  - IDE에서 브레이크포인트를 찍어도, 람다 안의 지역 변수가 t 하나뿐이라 흐름을 따라가기 어렵습니다.
  - 스트림 내부에서 제네릭 타입 추론이 복잡하게 얽히므로 IDE의 타입 추론이 제한적입니다.
  - 선언형 코드가 의도를 숨길 수 있습니다.
- 따라서, 간단한 EnumMap 초기화에는 기존의 반복문 방식이 더 직관적일 수 있습니다.

## 3. 결론

- `ordinal()` 은 간단하지만 타입 안전성과 유지보수성이 떨어집니다.
- EnumMap 은 배열 수준의 성능과 Map의 가동성, 안전성을 모두 제공합니다.
- 단순한 경우에는 for 루프, 복잡한 경우에는 스트림을 사용하는 등 목적에 맞게 선택하여 활용할 수 있습니다.

---

### 질문

Q. `ordinal()` 메서드를 이용해 배열 인덱스로 접근하는 방식의 문제점은 무엇인가요?

Q. EnumMap은 내부적으로 배열을 사용하면서도 타입 안전성을 유지할 수 있는 이유는 무엇인가요?

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 37