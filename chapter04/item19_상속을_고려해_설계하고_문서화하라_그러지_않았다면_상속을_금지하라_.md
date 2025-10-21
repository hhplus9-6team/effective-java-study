# item19. 상속을 고려해 설계하고 문서화하라. 그러지 않았다면 상속을 금지하라

## 1. 상속의 위험성과 배경

```java
class Parent {
    void print() { System.out.println("Parent"); }
}

class Child extends Parent {
    @Override
    void print() {
        super.print(); // ✅ 부모 메서드 호출
        System.out.println("Child");
    }
}
```

상속은 객체지향의 대표적인 재사용 메커니즘이지만, 부모 클래스의 내부 구현 세부사항에 하위 클래스가 깊이 얽히는 문제를 야기합니다.

부모가 내부 동작을 바꾸면, 이를 상속한 하위 클래스가 예상치 못하게 깨질 수 있습니다.

따라서, 상속은 캡슐화를 약화시키며, 유지보수를 어렵게 만들 수 있습니다.

**상속을 허용하려면, 클래스의 내부 호출 순서와 재정의 가능한 메서드의 호출 시점을 반드시 문서로 명시해야 합니다.**

## 2. 문서적 명시 : `@implSpec` 태그

```java
    /**
     * {@inheritDoc}
     *
     * @implSpec
     * This implementation iterates over the collection looking for the
     * specified element.  If it finds the element, it removes the element
     * from the collection using the iterator's remove method.
     *
     * <p>Note that this implementation throws an
     * {@code UnsupportedOperationException} if the iterator returned by this
     * collection's iterator method does not implement the {@code remove}
     * method and this collection contains the specified object.
     *
     * @throws UnsupportedOperationException {@inheritDoc}
     * @throws ClassCastException            {@inheritDoc}
     * @throws NullPointerException          {@inheritDoc}
     */
    public boolean remove(Object o) {...}
```

- 상속을 고려한 클래스는 "이 메서드가 내부적으로 어떻게 동작하는지, 하위 클래스가 지켜야 할 규약이 무엇인지"를 Javadoc의 `@implSpec` 태그로 명시해야 합니다.

- `@implSpec`은 컴파일 타임에 아무 동작도 하지 않고, 오직 Javadoc 생성 시 문서화용으로만 사용됩니다.
    ```
    javadoc -d doc -sourcepath src com.example
    ```


- `javadoc` 명령어 실행 시, `@implSpec` 블록은 "Implementation Requirements" 섹션으로 렌더링됩니다.
    > Implementation Requirements: The default implementation obtains an iterator over the Iterable and invokes the given action...
    - 예시 : [Iterable#forEach (Java SE 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Iterable.html#forEach(java.util.function.Consumer)), [Collection#removeIf (Java SE 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Collection.html#removeIf(java.util.function.Predicate)), [Map#forEach (Java SE 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Map.html#forEach(java.util.function.BiConsumer))

- `javadoc` 명령어 실행 시, `-tag "implSpec:a:Implementation Requirements:"` 옵션을 추가해야 `@implSpec` 태그 활성화가 가능합니다.
  - `@implSpec`은 비표준 Javadoc 태그로 Java 언어 문법의 일부가 아니라 문서 생성 시 특정 플래그를 켜야만 해석되는 주석 태그이기 때문입니다.

## 3. 효율적인 확장 : `protected` 메서드와 `protected` 필드 사용

### `protected` 메서드

```java
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {
    
    ...

    public void clear() {
        removeRange(0, size());
    }

    protected void removeRange(int fromIndex, int toIndex) {
        ListIterator<E> it = listIterator(fromIndex);
        for (int i=0, n=toIndex-fromIndex; i<n; i++) {
            it.next();
            it.remove();
        }
    }
}
```
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    
    public void clear() {
        modCount++;
        final Object[] es = elementData;
        for (int to = size, i = size = 0; i < to; i++)
            es[i] = null;
    }

    protected void removeRange(int fromIndex, int toIndex) {
        if (fromIndex > toIndex) {
            throw new IndexOutOfBoundsException(
                    outOfBoundsMsg(fromIndex, toIndex));
        }
        modCount++;
        shiftTailOverGap(elementData, fromIndex, toIndex);
    }

    private void shiftTailOverGap(Object[] es, int lo, int hi) {
        System.arraycopy(es, hi, es, lo, size - hi);
        for (int to = size, i = (size -= hi - lo); i < to; i++)
            es[i] = null;
    }
}
```

- 상속을 고려한 클래스는 하위 클래스가 확장할 수 있도록 적절한 훅(hook) 메서드를 protected 형태로 제공해야 합니다.

- 예시를 보면, removeRange()가 protected로 열려 있기 때문에, 하위 클래스가 불필요한 전체 순회를 피하고 효율적으로 동작하도록 돕습니다.

### `protected` 필드

```java
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {
    protected transient int modCount = 0;
}
```
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    ...

    public void clear() {
        modCount++;
        final Object[] es = elementData;
        for (int to = size, i = size = 0; i < to; i++)
            es[i] = null;
    }
}
```

- 상속 클래스가 내부 변경을 감지할 수 있도록 modCount 같은 필드를 protected로 공개하는 경우도 있습니다.

- 예시를 보면, 하위 클래스가 구조를 변경할 때(예: add/remove) 기존 반복자의 동작을 깨뜨리지 않도록 관리하는 데 사용됩니다.

## 4. 상속용 클래스 테스트

상속을 고려한 클래스는 반드시 실제로 하위 클래스를 만들어 동작을 검증해야 합니다.

그래야 안전한 초기화, 호출 순서, protected 훅이 올바르게 작동하는지 확인할 수 있습니다.

단순히 컴파일만 통과하는 게 아니라, 하위 클래스가 재정의했을 때 예상치 못한 부작용 없이 정상 동작하는지 반드시 테스트해야 합니다.

## 5. 상속 시 유의 사항

### 1) 생성자에서 재정의 가능한 메서드 호출 금지

```java
class Base {
    Base() {
        override(); // ❌ 아직 하위 클래스 초기화 전
    }
    void override() {}
}

class Sub extends Base {
    private final int value;

    Sub(int v) {
        value = v;
    }

    @Override
    void override() {
        System.out.println(value); // ❌ value는 아직 0 (미초기화)
    }
}

Base base = new Sub(1);
```

생성자 안에서 재정의 가능한 메서드를 호출하면,  하위 클래스의 필드 초기화 이전에 호출되어 의도치 않은 동작이 발생할 수 있습니다.


1. Sub(1) 생성자를 호출하면, 자바는 항상 부모 생성자부터 호출합니다. 즉, Base() 생성자가 먼저 실행됩니다.

2. Base() 생성자 내부에서 override() 를 호출합니다.

3. 그런데 override()는 Sub 클래스에서 재정의(override) 되어 있으므로, 실제로 호출되는 건 Sub.override() 입니다.

4. 하지만 이 시점에서는 Sub의 필드가 아직 초기화되지 않습니다. Sub 생성자의 본문(value = v;)은 아직 실행되지 않은 상태입니다. 그래서 value는 여전히 기본값 0 인 상태로 남아있습니다.

5. 따라서 출력은 다음과 같습니다 : `0`

### 2) `clone()`과 `readObject()`도 동일 원칙 적용

객체 복제(clone)나 역직렬화(readObject) 도 객체가 완전히 초기화되기 전 단계에서 호출됩니다.

따라서 이 메서드 내부에서도 재정의 가능한 메서드를 호출해서는 안 됩니다.

### 3) `Serializable`, `Cloneable`은 상속 설계를 복잡하게 만듦

이 두 인터페이스를 부모 클래스가 구현해버리면, 하위 클래스가 의도치 않게 직렬화 대상, 복제 가능 객체가 되어버립니다.

따라서 하위 클래스가 필요하다면 스스로 구현하게 두는 것이 안전합니다.

예외적으로 `readResolve`, `writeReplace` 메서드는 하위 클래스가 재정의할 수 있도록 protected로 선언해야 합니다. (private이면 하위 클래스에서 가려져 무시됨)

- JVM에서는 직렬화/역직렬화 직전/직후에 해당 메서드가 있는지 확인한  호출하고 이 때 싱글턴 객체로 바꿀 수 있는 것인데, private 로 선언하게 되면 하위 클래스에 존재하지 않는 것으로 인식하게 되어 무시됩니다.

## 6. 결론

상속은 “문서 + 테스트 + 계약 기반 확장” 이 전제되어야만 안전합니다.

상속을 고려하지 않은 클래스라면 final 선언하거나 컴포지션(composition) 으로 대체하는 것이 좋습니다.

---

### 질문

Q. 다음 중 올바르게 상속을 고려한 클래스 설계는 무엇인가요?

A) 하위 클래스가 어떻게 재정의하든 상관없게 private 메서드에서 모든 로직을 처리한다.
B) 하위 클래스가 규약을 지켜야 할 부분은 @implSpec으로 문서화하고, 필요시 protected 훅 메서드를 제공한다.
C) 하위 클래스가 재정의할 때 내부 필드에 직접 접근할 수 있도록 전부 public 으로 만든다.

Q. 아래 코드를 실행했을 때 콘솔에는 어떤 값이 출력되며, 왜 이런 결과가 발생하나요?

```java
class Base {
    Base() {
        override();
    }
    void override() {}
}

class Sub extends Base {
    private final int value;

    Sub(int v) {
        value = v;
    }

    @Override
    void override() {
        System.out.println(value);
    }

    public static void main(String[] args) {
        new Sub(42);
    }
}
```

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 19
- [Iterable#forEach (Java SE 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Iterable.html#forEach(java.util.function.Consumer))
- [Collection#removeIf (Java SE 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Collection.html#removeIf(java.util.function.Predicate))
- [Map#forEach (Java SE 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Map.html#forEach(java.util.function.BiConsumer))