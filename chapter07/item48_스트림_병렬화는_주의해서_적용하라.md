# item 48. 스트림 병렬화는 주의해서 사용하라
> “병렬 스트림은 강력하지만, 올바른 조건에서만 효과적이다.
> 잘못 사용하면 결과가 틀리거나 오히려 성능이 떨어질 수 있다.”


---

## 1. 사전 지식: 병렬화와 동시성의 이해

### 1.1 병렬(Parallel) vs 동시(Concurrent)

| 구분                 | 설명                              | 예시                        |
| ------------------ | ------------------------------- | ------------------------- |
| **병렬(Parallel)**   | 여러 **CPU 코어가 실제로 동시에 실행**하는 것   | 8코어 CPU가 8개의 연산을 동시에 처리   |
| **동시(Concurrent)** | 하나의 CPU가 여러 스레드를 **교대로 실행**하는 것 | 싱글 코어 CPU가 2개 스레드를 번갈아 처리 |

> 즉, 동시성은 “겹쳐 보이는 실행”, 병렬성은 “진짜 동시에 실행”.

---

### 1.2 스레드와 CPU의 관계

| 구성                     | 설명                                                    |
| ---------------------- | ----------------------------------------------------- |
| **Java Thread**        | 개발자가 만든 스레드 (`Thread`, `@Async`, `ExecutorService` 등) |
| **OS Thread (커널 스레드)** | JVM이 운영체제에 등록하는 실제 실행 단위                              |
| **CPU Core**           | 물리적으로 동시에 연산 가능한 단위 (보통 4~16개)                        |

* 자바 스레드는 **OS 스레드에 1:1 매핑**됨
* OS가 이 스레드들을 **시분할 방식(time-slicing)** 으로 CPU에 배분
* CPU 코어 수보다 많은 스레드가 있으면, **컨텍스트 스위칭 비용**이 발생

> 예: 8코어 CPU, 스레드 32개 → 실제 병렬 실행은 최대 8개
> 나머지는 OS 스케줄러가 번갈아가며 실행시킴

---

### 1.3 ThreadPool (스레드 풀)

스레드를 매번 생성/삭제하는 비용을 줄이기 위해,
미리 스레드들을 만들어두고 재사용하는 구조.

#### 자바의 스레드풀

* `ExecutorService`, `ThreadPoolExecutor`, `ForkJoinPool`
* **JDK 레벨**의 스레드풀

#### 스프링의 스레드풀

* `ThreadPoolTaskExecutor`, `@Async`, `@Scheduled`
* 내부적으로 **Java ThreadPoolExecutor를 래핑**
* 결국 OS 스레드를 공유함

> 스레드풀을 여러 개 두더라도 CPU는 물리적으로 동시에 하나의 스레드만 코어당 실행한다.
> 여러 풀은 단지 “작업 큐”를 분리해 관리할 뿐이다.

---

### 1.4 병렬 스트림(Parallel Stream)과 ForkJoinPool

* `stream()` → 순차 실행 (한 스레드)
* `parallelStream()` → 병렬 실행 (여러 스레드)
* 내부적으로 `ForkJoinPool.commonPool()`을 사용

```java
int sum = numbers.parallelStream()
        .mapToInt(Integer::intValue)
        .sum();
```

이 한 줄의 내부 동작은,
데이터를 여러 **“청크(chunk)”** 로 나누고, 각 청크를 다른 스레드가 처리한다.

| 스레드 | 데이터 처리 구간           |
| --- | ------------------- |
| T1  | 1 ~ 250,000         |
| T2  | 250,001 ~ 500,000   |
| T3  | 500,001 ~ 750,000   |
| T4  | 750,001 ~ 1,000,000 |

> ForkJoinPool도 결국 **OS 스레드 기반**이므로 CPU 코어 수를 넘는 병렬성은 불가능.

### 1.5. 병렬 스트림의 동작 방식
#### 1.5.1. 스트림 파이프라인 구조

스트림은 다음 세 단계로 구성된다.

1. **소스(Source)**: 데이터를 제공하는 컬렉션, 배열, 범위 등
2. **중간 연산(Intermediate Operation)**: `map`, `filter`, `sorted`, `distinct` 등.
   이 단계는 **지연(lazy)** 되어 있으며, 즉시 실행되지 않는다.
3. **최종 연산(Terminal Operation)**: `forEach`, `collect`, `reduce` 등.
   이 시점에서 실제 데이터 처리가 일어난다.

---

#### 1.5.2. 중간 연산의 실행 순서 (순차 스트림)

스트림은 중간 연산을 한 단계씩 모두 수행한 뒤 다음 단계로 넘어가는 방식이 아니다.
각 요소(element)가 파이프라인 전체(`map → filter → ... → forEach`)를 통과하면서 처리된다.

예를 들어 다음과 같은 코드가 있을 때:

```java
List.of(1, 2, 3, 4).stream()
    .map(x -> { System.out.println("map " + x); return x + 1; })
    .filter(x -> { System.out.println("filter " + x); return x % 2 == 0; })
    .forEach(x -> System.out.println("forEach " + x));
```

출력 순서는 다음과 같다.

```
map 1
filter 2
forEach 2
map 2
filter 3
map 3
filter 4
forEach 4
map 4
filter 5
```

즉, **각 요소가 한 번에 전체 파이프라인을 통과한다.**
중간 연산 전체를 마친 뒤 다음 연산으로 넘어가는 구조가 아니다.

---

#### 1.5.3. 병렬 스트림의 실행 순서

`parallelStream()`을 사용하면 각 요소가 **여러 스레드에 분산되어 병렬로 처리**된다.
다만, 여전히 각 요소는 `map → filter → forEach` 순으로 파이프라인을 통과한다.

```java
List.of(1, 2, 3, 4).parallelStream()
    .map(x -> { System.out.println("map " + x + " : " + Thread.currentThread().getName()); return x + 1; })
    .filter(x -> { System.out.println("filter " + x + " : " + Thread.currentThread().getName()); return x % 2 == 0; })
    .forEach(x -> System.out.println("forEach " + x + " : " + Thread.currentThread().getName()));
```

출력 예시:

```
map 1 : ForkJoinPool.commonPool-worker-3
map 3 : ForkJoinPool.commonPool-worker-5
map 2 : ForkJoinPool.commonPool-worker-2
map 4 : ForkJoinPool.commonPool-worker-1
filter 2 : ForkJoinPool.commonPool-worker-3
filter 4 : ForkJoinPool.commonPool-worker-2
forEach 2 : ForkJoinPool.commonPool-worker-3
forEach 4 : ForkJoinPool.commonPool-worker-2
```

이처럼 병렬 스트림은

* 각 요소 단위로 파이프라인이 수행되며
* 여러 스레드가 서로 다른 요소를 동시에 처리한다.
* `map`, `filter` 등 중간 연산들이 **병렬로 실행되는 것처럼 보이지만**, 실제로는 **각 스레드가 자신에게 할당된 데이터 일부를 파이프라인 전체로 처리**하는 방식이다.

---

#### 1.5.4. 내부 동작 요약

1. **데이터 분할(Split)**

    * 소스 데이터를 여러 조각(chunk)으로 나눔
    * 각 조각은 개별 스레드에서 처리됨

2. **병렬 처리(Fork)**

    * ForkJoinPool의 워커 스레드들이 각 청크를 담당
    * 각 스레드는 `map → filter → ...` 연산을 순서대로 수행

3. **결과 병합(Join)**

    * 최종 연산(`reduce`, `collect` 등)에서 부분 결과를 합침

---

## 2. 스트림 병렬화

### 2.1 병렬 스트림의 의도

스트림 API는 **함수형 스타일로 데이터 처리**를 단순화했지만,
`.parallel()`을 붙이면 쉽게 병렬 실행이 가능하다는 점 때문에 오용이 많다.

```java
// 순차 스트림
long sum = LongStream.rangeClosed(1, n).reduce(0, Long::sum);

// 병렬 스트림 (겉보기엔 동일)
long sum = LongStream.rangeClosed(1, n).parallel().reduce(0, Long::sum);
```

이 두 코드가 **항상 같은 결과**나 **더 나은 성능**을 보장하지 않는다.

---

### 2.2 병렬 스트림의 위험성

1. **결과의 정확성이 깨질 수 있음**

    * 연산이 비결정적(non-deterministic)일 경우
      → 순서 보장, 상태 공유, 의존성 문제 등

2. **성능이 오히려 나빠질 수 있음**

    * 병렬화 오버헤드 (스레드 생성, 컨텍스트 스위칭)
    * 데이터 분할이 비효율적일 경우
    * I/O 중심 로직일 경우

3. **공유 상태를 변경하면 안 됨**

    * 병렬 스트림 안에서는 외부 상태를 변경하지 않아야 함 (side effect 금지)

예:

```java
List<Integer> list = new ArrayList<>();
IntStream.range(0, 1000).parallel().forEach(list::add); // ❌ 경쟁 상태 발생
```

---

### 2.3 병렬화가 유효한 조건

| 조건                         | 설명                                                |
| -------------------------- | ------------------------------------------------- |
| **1. 연산이 CPU 중심이고, 비용이 큼** | 코어를 나누어 계산할 가치가 있을 때                              |
| **2. 데이터 간 의존성이 없음**       | 각 원소가 독립적으로 처리될 때                                 |
| **3. 데이터 소스가 분할 가능**       | `ArrayList`, `IntStream.range`, `Arrays.stream` 등 |
| **4. 연산 순서가 중요하지 않음**      | reduce, sum 등 순서 의존이 없는 연산                        |

---

### 2.4 병렬화가 불리한 조건

| 조건                   | 설명                                    |
| -------------------- | ------------------------------------- |
| **I/O 연산 포함**        | 네트워크, DB 접근은 스레드 병렬화 효과 거의 없음         |
| **데이터가 작거나 연산이 가벼움** | 병렬화 오버헤드가 더 큼                         |
| **소스가 분할 어려움**       | `LinkedList`, `Stream.iterate()` 등    |
| **순서가 중요한 연산**       | `forEachOrdered`, `sorted`, `limit` 등 |

---

### 2.5 올바른 활용 예시

```java
// 예시: 큰 범위의 수에서 소수 개수 계산
long count = LongStream.rangeClosed(2, 10_000_000)
                       .parallel()
                       .filter(Item48::isPrime)
                       .count();
```

* 데이터는 수치형 범위(IntStream)
* 각 원소는 독립적 (isPrime 계산)
* 연산 비용이 큼 → 병렬화 이점이 발생

---

### 2.6 실전 조언

1. **무작정 `.parallel()`을 붙이지 말라.**
   → 오히려 느려질 수 있다.

2. **병렬화 전후 성능을 측정하라.**
   → 벤치마크/프로파일링 필수.

3. **공유 상태(side effect) 없이 순수 함수만 사용하라.**

4. **적합한 데이터 소스를 선택하라.**

5. **CPU 코어 수 이상으로 병렬화는 의미 없다.**
   → CPU 바운드 작업에만 적합.