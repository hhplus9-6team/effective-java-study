# item11. equals를 재정의하려거든 hashCode도 재정의하라

**equals를 재정의한 클래스는 hashCode도 재정의 해야 한다.**  
그렇지 않으면 인스턴스를 HashMap이나 HashSet 같은 컬렉션의 원소로 사용할 때 문제가 발생한다.

<details><summary>hashCode란?</summary>

- 객체를 구분하는 정수 값 (식별자 역할)
- 주 사용처: 해시테이블 기반 컬렉션(Hashtable, HashMap, HashSet 등)
- Object의 기본 구현: **객체의 메모리 주소를 기반**으로 한 해시코드 반환
- 객체가 다르면 보통 해시코드도 다르지만, 반드시 그렇지는 않음 (충돌 가능성 존재)

**사용 시점**

1. 해시 기반 컬렉션
    - HashMap, HashSet, Hashtable 같은 자료구조는 내부적으로 버킷(bucket)을 사용해 데이터를 저장한다.
    - 객체를 저장하거나 검색할 때 먼저 hashCode로 버킷 위치를 찾고, 그 다음에 equals로 같은 객체인지 확인한다.
    - 즉, hashCode는 **빠른 분류**, equals는 **정확한 판별**을 담당한다.

2. 중복 검사
    - HashSet에 객체를 추가할 때)
        - hashCode가 다르면 equals 비교도 하지 않고 다른 객체로 취급한다.
        - hashCode가 같으면 equals를 확인해 최종적으로 같은 객체인지 판정한다.

3. 성능 최적화
    - 선형 탐색(O(n)) 대신 해시코드를 이용해 평균적으로 O(1)에 가까운 속도로 데이터를 찾을 수 있다.
    - hashCode를 잘못 구현하면 해시 충돌이 늘어나 성능이 급격히 나빠질 수 있다.

</details>

## hashCode 규약

1. equals 비교 정보가 바뀌지 않았다면 실행 중에는 항상 같은 hashCode를 반환해야 한다.   
  (단, 애플리케이션을 다시 실행한다면 이 값이 달라져도 상관없다.)
2. equals가 두 객체를 같다고 판단하면 반드시 같은 hashCode를 반환해야 한다.
3. equals가 두 객체를 다르다고 판단했더라도 서로 다른 hashCode를 반환할 필요는 없다. 하지만 해시테이블 성능을 위해 다른 객체에 대해서 다른 값을 반환하는게 좋다.

> [!IMPORTANT]
> hashCode를 잘못 정의했을 때 **문제가 되는 건 2번 규약**이다.
> - 논리적으로 같은 객체라면 **반드시** 같은 해시코드를 반환해야 한다.
> - 하지만 기본 Object.hashCode()는 물리적으로 같은 객체만 같은 해시코드를 반환하므로 equals만 재정의하고 hashCode를 재정의하지 않으면 논리적으로 같은 두 객체가 서로 다른 해시코드를
    갖게 된다. (equals는 물리적으로 다른 두 객체를 논리적으로 같다고 판단할 수 있다.)

## 잘못된 hashCode 예

```java
public class HashCode {
    public static void main(String[] args) {
        Map<PhoneNumber, String> m = new HashMap<>();
        m.put(new PhoneNumber(010, 1234, 5678), "leevigong");

        String result = m.get(new PhoneNumber(010, 1234, 5678));
        System.out.println("result = " + result);

        System.out.println("new PhoneNumber(010, 1234, 5678).hashCode() = " + new PhoneNumber(010, 1234, 5678).hashCode());
        System.out.println("new PhoneNumber(010, 1234, 5678).hashCode() = " + new PhoneNumber(010, 1234, 5678).hashCode());
    }

}

class PhoneNumber {
    private final int areaCode;
    private final int prefix;
    private final int lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = areaCode;
        this.prefix = prefix;
        this.lineNum = lineNum;
    }
}
```

실행결과:

```
result = null
new PhoneNumber(010, 1234, 5678).hashCode() = 1915910607
new PhoneNumber(010, 1234, 5678).hashCode() = 317574433
```
- PhoneNumber는 hashCode를 재정의하지 않았기 때문에, 논리적 동치인 두 객체가 서로 다른 해시코드를 반환한다.
- HashMap은 해시코드로 먼저 버킷을 찾고, 같은 버킷에서 equals로 비교한다. 해시코드가 다르면 equals는 호출조차 되지 않는다. 따라서 m.get(...)은 null을 반환한다.

## 해결법: hashCode 재정의하기

### 최악의 구현 (상수 반환)

```java

@Override
public int hashCode() {
    return 42;
}
```

- 모든 인스턴스에 대해 같은 해시코드를 반환한다.
- 규약 2번은 만족하지만, 해시테이블 성능이 매우 나빠진다.
    - 모든 객체에게 똑같은 값만 내어주므로 모든 엔트리가 같은 버킷에 들어가게 되어, 해시테이블이 아니라 연결리스트가 된다.
    - 그 결과, 평균 수행시간이 O(1)에서 O(n)으로 악화된다.

---
**좋은 해시 함수라면 서로 다른 인스턴스에 다른 해시코드를 반환한다.**

좋은 hashCode를 작성하는 요령
1. int 변수인 result를 선언한 후 값을 c로 초기화한다.
    - c는 해당 객체의 첫번째 핵심 필드를 단계 2.1 방식으로 계산한 해시코드이다.
    - 여기서 핵심 필드는 equals 비교에 사용되는 필드를 말한다.
2. 해당 객체의 나머지 핵심 필드인 f 각각에 대해 다음 작업을 수행한다.
    1. 해당 필드의 해시코드 c를 계산한다.
        - 기본 타입 필드라면, Type.hashCode(f)를 수행한다. 
          - Type은 해당 기본타입의 박싱 클래스다.
        - 참조 타입 필드면서 이 클래스의 equals 메소드가 이 필드의 equals를 재귀적으로 호출하여 비교한다면, 이 필드의 hashCode를 재귀적으로 호출한다.  
          - 계산이 더 복잡해질 것 같으면 이 필드의 표준형을 만들어 그 표준형의 hashCode를 호출
          - 참조 타입 필드가 null일 경우 0을 사용
        - 필드가 배열이라면, 핵심 원소 각각을 별도 필드처럼 다룬다.
          - 이상의 규칙을 재귀적으로 적용해 각 핵심 원소의 해시코드를 계산한 다음 2.b 방식으로 갱신
          - 핵심 원소가 하나도 없다면 단순 상수(0 추천)를 사용
          - 모든 원소가 핵심 원소라면 Arrays.hashCode를 사용
    2. 단계 2.1에서 계산한 해시코드 c로 result를 갱신한다.
       - result = 31 * result + c;
3. result를 반환한다.

> **31을 곱하는 이유?**  
> 31은 **소수이면서 홀수**이기 때문에 선택된 값이다.  
> 만약 짝수를 곱했다면 곱셈 결과가 오버플로될 때 정보 손실이 생길 수 있다.  
> (예: 2를 곱하는 것은 단순 시프트 연산과 같아 해시 분포가 고르지 않을 수 있음)
>
> 소수를 쓰는 것 자체의 이점은 뚜렷하지 않지만, 전통적으로 널리 사용되어 왔다.  
> 특히 31은 `(i << 5) - i`로 최적화할 수 있어서 계산 효율성이 높다.  
> 최신 JVM에서는 이런 최적화를 자동으로 수행한다.

---

### 👍🏻 전형적인 hashCode 메서드 구현

```java

@Override
public int hashCode() {
    int result = Integer.hashCode(areaCode);
    result = 31 * result + Integer.hashCode(prefix);
    result = 31 * result + Integer.hashCode(lineNum);
    return result;
}
```


### 단순 구현 (Objects.hash())
Objects 클래스는 임의의 개수만큼 객체를 받아 해시 코드를 계산해주는 정적 메서드 hash를 제공한다.  
이 코드는 한 줄로 작성되지만 성능이 떨어진다.
- 내부적으로 입력 인수를 담기 위한 배열 생성하고 기본 타입이 있다면 박싱과 언박싱 발생하기 떄문에
```java

@Override
public int hashCode() {
    return Objects.hash(areaCode, prefix, lineNum);
}
```

### 캐싱 구현 (지연 초기화 전략)
- 클래스가 불변이고 해시코드를 계산하는 비용이 클 때 성능면에서 유용
- 필드를 지연 초기화하려면 그 클래스를 스레드 안전성을 고려해야 한다
```java
private int hashCode; // 기본값 0

@Override
public int hashCode() {
    int result = hashCode;
    if (result == 0) {
        result = Integer.hashCode(areaCode);
        result = 31 * result + Integer.hashCode(prefix);
        result = 31 * result + Integer.hashCode(lineNum);
        hashCode = result;
    }
    return result;
}
```
** hashCode 필드 초기값은 흔히 생성되는 객체 해시 코드와 달라야 한다. (보통 0)

### 주의사항
- equals 비교에 사용되지 않는 필드는 반드시 제외해야한다.
- 핵심 필드를 생략해서는 안된다
  - 빠른 성능을 노리고 필드를 빼면 속도는 빠를 수 있지만, 해시 충돌이 많이 발생해 해시테이블 성능이 오히려 떨어진다.
  - 예) 자바 2 이전의 String은 최대 16개의 문자만으로 해시코드 계산, 문자열이 길면 균일하게 나눠 16 문자만 뽑아내 사용했다;;
- hashCode가 반환하는 값의 생성 규칙을 API 사용자에게 자세히 공표하지 말자. 그래야 클라이언트가 이 값에 의지하지 않게 되고, 추후에 계산 방식을 바꿀 수도 있다.


## 결론

equals를 재정의하면 반드시 hashCode도 재정의해야 한다. 그렇지 않으면 프로그램이 제대로 동작하지 않을 것 이다.
재정의한 hashCode는 Object의 API 문서에 기술된 일반 규약을 따라야 하며, 서로 다른 인스턴스라면 되도록 해시코드도 서로 다르게 구현해야 한다.   
AutoValue, Lombok를 사용하면 멋진 equals와 hashCode를 자동으로 만들어준다.


---

## 질문

Q1. equals를 재정의할 때 hashCode도 반드시 함께 재정의해야 하는 이유는 무엇인가요?  
Q2. 해시 함수 구현 시 31을 곱하는 방식이 권장되는 이유는 무엇인가요?  
Q3. 불변 객체에서 hashCode를 지연 초기화(캐싱)하면 어떤 장점이 있을까요?
