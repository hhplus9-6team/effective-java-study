# item 58. 전통적인 for문보다는 for-each문을 사용하라
> * **가능한 모든 곳에서 for-each를 사용하라.**
> * 반복자를 직접 사용해야 하는 극히 일부 상황(삭제, 병렬 순회 등)에서만 전통적 for문을 쓰면 된다.
> * for-each는 컬렉션, 배열, Iterable을 다루는 기본 패턴이며,
  **효율 + 안정성 + 가독성 세 마리 토끼를 모두 잡는 루프 방식**이다.

## 1. 개요

* 배열이나 컬렉션을 순회할 때는 전통적 for문 또는 while(iterator)보다
  for-each문(enhanced for문)이 더 간결하고 버그를 예방한다.
* for-each문은 반복자로 처리되는 모든 요소 순회를 자동으로 처리해 주기 때문에
  인덱스 오류, 반복자 처리 실수 등을 줄여준다.

---

## 2. 전통적 for문 / Iterator의 문제점

### - 명령형 코드가 복잡해진다

```java
for (Iterator<Element> i = list.iterator(); i.hasNext();) {
    Element e = i.next();
    ... // 작업
}
```

### - 반복자 사용 실수 가능

* `i.next()`를 너무 많이 호출하거나
* 반복자를 복사해 잘못 사용하거나
* 인덱스를 잘못 계산하는 등의 오류가 발생하기 쉽다.

### - 인덱스 기반 for는 컬렉션에 적합하지 않다

```java
for (int i = 0; i < list.size(); i++) {
    process(list.get(i));
}
```

* LinkedList처럼 인덱스 접근 비용이 큰 컬렉션은 성능 저하

---

## 3. for-each 문이 더 좋은 이유

### - 간결하고 가독성이 뛰어남

```java
for (Element e : list) {
    process(e);
}
```

### - 오류 가능성을 구조적으로 제거

* 인덱스 계산이 없다
* 반복자 관리가 없다
* next() 호출 타이밍을 실수할 위험도 없다

### - 컬렉션, 배열, Iterable 모두 동일한 문법

일관된 코드 스타일 유지 가능.

---

## 4. for-each 문을 사용할 수 없는 경우

for-each가 **모든 상황에 완전 대체 가능한 것은 아니다.**

### 1) **요소를 수정해야 할 때 (Iterator.remove 필요)**

ConcurrentModification 없이 요소 삭제/변경을 해야 하면 전통적인 반복이 필요.

```java
for (Iterator<Element> it = list.iterator(); it.hasNext();) {
    Element e = it.next();
    if (shouldRemove(e)) {
        it.remove(); // for-each에서는 불가능
    }
}
```

### 2) **병렬 반복이 필요한 경우**

두 리스트를 index 기반으로 동시에 순회할 때.

```java
for (int i = 0; i < list1.size(); i++) {
    doSomething(list1.get(i), list2.get(i));
}
```

### 3) **필요한 요소가 값이 아닌 index 그 자체일 때**

인덱스가 도메인 의미를 가질 때 (예: DP 테이블 인덱스, 좌표 등)

### 4) **반복 순서를 제어해야 할 때**

중간에 특정 조건에서 "다음 요소 2개 건너뛰기" 같은 흐름 제어가 필요하면 일반 반복문이 더 적합.

---

## 5. 성능 관련

* for-each문은 내부적으로 반복자를 사용하므로 성능 차이는 거의 없다.
* 배열의 경우에는 전통적 for문과 for-each문 사이에 사실상 차이 없음.
* CPU 캐시나 JIT 최적화 측면에서도 동일한 수준.
* 오히려 **가독성 + 안전성** 측면에서 항상 유리하다.

## QnA
- 전통적 for문의 단점은?