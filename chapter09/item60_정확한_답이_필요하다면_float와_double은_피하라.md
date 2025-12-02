# item60. 정확한 답이 필요하다면 float와 double은 피하라

## 1. 서론

자바 개발자가 흔히 저지르는 실수 중 하나가 `float`와 `double`을 정확한 수치 계산용으로 사용하는 것입니다.

하지만 이 타입들은 정밀한 값 표현을 위해 설계된 타입이 아닙니다.

특히 돈, 수량, 정산, 포인트 같은 도메인에서는 치명적인 오류를 만들 수 있습니다.

따라서, 정확한 결과가 필요하다면 `float`와 `double`을 사용하지 말아야 합니다.


## 2. 왜 `float`와 `double`은 정확하지 않을까?

### 1) 부동소수점 방식의 한계

자바에서 `float`와 `double`은 **IEEE 754 부동소수점 방식**으로 값을 저장합니다.

이 방식은 숫자를 **10진수가 아닌 2진수 기반**으로 숫자를 표현합니다.

문제는 우리가 일상적으로 사용하는 10진수 소수값들이 2진수로는 정확하게 표현이 불가능한 값이라는 점입니다.

특정 10진수 소수값들을 2진수로 변환할 때 무한히 반복되는 소수가 됩니다.

```java
System.out.println(1.03 - 0.42);

//출력 결과
0.6100000000000001
```

개발자가 기대한 값은 `0.61` 이지만 실제로는 아주 작은 오차가 포함된 값이 출력됩니다.

이 오차는 계산 과정에서 생기는 실수가 아니라, 애초에 `1.03`과 `0.42`가 정확하게 저장되지 못했기 때문입니다.

### 2) 왜 0.1 은 2진수로 무한 반복이 되는가?

핵심은 2진법이 표현할 수 있는 분수의 형태에 있습니다.

2진법에서 유한 소수가 되려면 분모가 반드시 **2의 거듭제곱(2ⁿ)** 이어야 합니다.

```
1 / 10 = 1 / (2 * 5)
```

2진법은 2의 거듭제곱으로만 표현할 수 있으므로 분모에 5가 포함된 순간 유한한 비트로 정확히 표현할 수 없게 됩니다.

```
0.1 × 2 = 0.2 → 0
0.2 × 2 = 0.4 → 0
0.4 × 2 = 0.8 → 0
0.8 × 2 = 1.6 → 1
0.6 × 2 = 1.2 → 1
0.2 × 2 = 0.4 → 0
… 반복
```

```
0.00011001100110011001100… (무한 반복)
```

따라서 0.1을 2진수 변환하게 되면 무한 반복되는 형태로 나타나게 됩니다.

컴퓨터는 유한한 비트 안에 이 값을 저장해야 하므로 중간에서 끊어서 가장 가까운 근사값으로 반올림합니다.

이 때 이미 작은 오차가 발생하는 것입니다.

### 3) IEEE 754

모든 프로그래밍 언어에서 사용하는 부동소수점 방식은 국제 표준인 **IEEE 754**를 따릅니다.

이 표준은 다음을 정의합니다.

- 실수를 저장하는 방식
- 부호 / 지수 / 가수 비트의 구조
- 반올림 방식
- 무한대, NaN 처리 방식
- 덧셈·뺄셈·곱셈·나눗셈 연산 규칙

이에 따라 `double` 은 다음과 같이 정해진 64비트 안에 실수를 근사하여 저장하는 방식입니다.

구성요소 | 비트 수
-|-
부호 (Sign) | 1비트
지수 (Exponent) | 11비트
가수 (Fraction) | 52비트

이렇게 저장된 근사값들끼리 연산을 반복하다보면 오차가 누적되어 정확한 결과에서 점점 더 멀어지게 됩니다.


## 2. 대안 : `BigDecimal`

정확한 계산이 필요하다면 `BigDecimal`을 사용해야 합니다.

`BigDecimal`은 이진 부동소수점 방식이 아닌 십진 기반으로 숫자를 표현하기 때문에 정확하게 값을 저장하고 연산할 수 있습니다.

```java
BigDecimal a = new BigDecimal("1.03");
BigDecimal b = new BigDecimal("0.42");

System.out.println(a.subtract(b)); // 0.61
```

또한 `BigDecimal`은 반올림 규칙까지 명확하게 제어할 수 있습니다.

```java
amount.multiply(rate)
      .setScale(2, RoundingMode.HALF_UP);
```

그래서 정산, 할인, 세금 계산 같은 도메인에서는 사실상 표준 방식입니다.

## 3. `BigDecimal` 실무 사용 팁

### 1) `BigDecimal` 비교는 `equals()`가 아니라 `compareTo()`

```java
new BigDecimal("1.0").equals(new BigDecimal("1"))   // false
```

- 1.0 → scale = 1
- 1 → scale = 0

`equals()`는 값 + 스케일을 함께 비교하기 때문에 값이 같아도 스케일이 다르면 false 가 됩니다.

```java
a.compareTo(b) == 0   // 같다
a.compareTo(b) > 0    // a가 더 크다
a.compareTo(b) < 0    // a가 더 작다

if (amount.compareTo(BigDecimal.ZERO) > 0) {
// 양수
}
```

따라서 `BigDecimal` 에서 비교가 필요할 때는 `compareTo()`를 사용해야 합니다.

### 2) 반올림은 `setScale()` + `RoundingMode` 로 명확히 지정

```java
value.setScale(2, RoundingMode.HALF_UP);
```

`BigDecimal`은 반올림 방식이 없으면 ArithmeticException을 던질 수 있습니다.

따라서 scale + roundingMode 를 항상 명확하게 지정해야 합니다.

#### 실무에서 자주 쓰는 RoundingMode

Mode | 설명 | 사용 예
-|-|-
HALF_UP | 0.5 이상 올림 | 가장 일반적인 상업용 반올림
HALF_EVEN | 은행식 반올림 (짝수 쪽으로) | 금융권/회계 기준
DOWN | 무조건 절삭 | 절삭형 정책
UP | 절대값 방향으로 올림 | 수수료 1월 단위 상

### 3) 생성자는 문자열 기반으로만

```java
new BigDecimal("1.03")  // O: 문자열 기반 → 정확한 값
new BigDecimal(1.03)    // X: 이미 double의 오차가 포함됨
```

`BigDecimal` 사용 시 문자열을 사용하지 않는 경우 오차가 포함될 수 있으니 주의가 필요합니다.

### 4) `BigDecimal` 상수는 반드시 `static final`로 재사용

```java
public static final BigDecimal ZERO = BigDecimal.ZERO;
public static final BigDecimal ONE = BigDecimal.ONE;
public static final BigDecimal HUNDRED = new BigDecimal("100");
```

`BigDecimal` 은 불변이라 상수로 선언해서 재사용하는 것이 좋습니다.

### 5) 금액 계산 시 반드시 “단위”를 고려하라

```java
price.multiply(new BigDecimal("0.1"));
```

이런식으로 사용하게 되면 스케일/표현 문제로 실수하기 쉽습니다.

```java
price.multiply(discountRate).divide(HUNDRED);

price.multiply(new BigDecimal("10"))
        .divide(new BigDecimal("100"), 2, RoundingMode.HALF_UP);
```

위와 같이 단위를 명확하게 표현하면 정책 변경에도 유연하고 오차 리스크도 거의 없어집니다.

### 6) 금액 누적 시 반드시 누적 변수도 BigDecimal로

```java
BigDecimal total = BigDecimal.ZERO;

for (Item item : items) {
    total = total.add(item.getPrice());
}
```

`double`을 혼합해서 더하면 오차가 그대로 누적될 수 있습니다.

### 7) 금액 비교 시 ZERO 기준 비교가 표준

```java
if (amount.compareTo(BigDecimal.ZERO) < 0) {
    // 음수일 때
}
```

### 8) BigDecimal → int/long 변환 시 절대로 silent cast 하지 말기

```java
int v = amount.intValue();
```

위와 같이 사용하게 되면 소수점이 버려지고 값 손실이 발생할 수 있습니다.

```java
amount.intValueExact();  // 소수점 있으면 예외 발생 → 안전
amount.longValueExact();
```

따라서 이와 같이 안전 변환을 하여야 금액 계산 시 정확성을 유지할 수 있습니다.

### 9) BigDecimal 연산 체인에서는 꼭 마지막에 scale 조절

```java
BigDecimal result = price
    .multiply(rate)
    .divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);
```

연산 후 마지막 단계에서 `setScale()`로 확정하는 것이 가장 안전합니다.

### 10) BigDecimal은 불변(immutable) → 반드시 재할당 필요

```java
price.add(fee); // ❌ 아무 효과 없음

price = price.add(fee); // ✅ 올바른 사용
```

실수로 `add()`, `multiply()` 호출만 하고 변수 재할당을 하지 않으면 버그로 이어질 수 있습니다.

## 4. 결론

`float`와 `double`은 빠르고 범위도 넓지만 정확한 계산을 보장하는 타입은 아닙니다.

이진수 기반의 부동소수점 방식은 여러 10진수 값을 정확하게 표현할 수 없고, 저장이나 연산 과정에서 오차가 누적되는 특성이 있습니다.

따라서 오차가 허용되지 않는 금액, 포인트, 수량 계산에서는 BigDecimal 또는 정수 타입을 사용해야 합니다.

---

### 질문

Q. 왜 정확한 계산에는 `float` / `double` 대신 `BigDecimal`을 사용해야 하나요?

Q. 왜 `new BigDecimal(1.03)`은 위험한 방식인가요?

Q. `BigDecimal`에서 `equals()`와 `compareTo()`의 차이를 설명해보세요.

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 60