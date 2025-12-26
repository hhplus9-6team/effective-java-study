# item75. 예외의 상세 메시지에 실패 관련 정보를 담으라

### 1. 왜 예외 메시지가 중요한가
- 예외가 잡히지 않고 프로그램이 종료되면, 스택 트레이스가 유일한 단서가 되는 경우가 많다.
- 스택 트레이스의 핵심은 결국 Throwable.toString() 결과이며,
  - 예외 클래스 이름 + 상세 메시지(detail message) 로 구성된다.
- 실패 재현이 어렵거나 운영 환경 문제일수록, 예외 메시지의 품질이 곧 디버깅 가능성을 좌우한다.

### 2. 상세 메시지에 반드시 담아야 할 정보
- 실패에 직접 관여한 모든 매개변수 값
- 실패 시점의 주요 필드 상태

예시: IndexOutOfBoundsException
- 범위의 최솟값(min)
- 범위의 최댓값(max)
- 실제로 접근한 index 값
```
Index out of range: index=12, valid range=[0, 9]
```

### 3. 장황할 필요는 없다.
- 관련 정보는 모두 담되, 불필요한 설명은 제거
- 개발자는 예외 메시지, 스택 트레이스, 소스 코드를 함께 보기 때문에, 메시지는 핵심 정보 위주로 작성하면 충분하다.

### 4. 예외 메시지 vs 사용자 오류 메시지
#### 예외의 상세 메시지
- 개발자 대상
- 가독성보다 정보 밀도가 중요
- 현지화(i18n) 대상이 아님

#### 사용자 오류 메시지
- 친절하고 이해하기 쉬워야 함
- 필요하다면 별도 계층에서 변환하여 제공

### 5. 생성자에서 실패 정보를 모두 받는 설계
- 예외 생성 시점에 필요한 정보를 모두 전달받아 상세 메시지를 즉시 완성하는 방식이 바람직하다.
```
public class InvalidRangeException extends RuntimeException {
  public InvalidRangeException(int index, int min, int max) {
  super("index=" + index + ", valid range=[" + min + ", " + max + "]");
  }
}
```

### 6. 접근자 메서드를 제공하라
- 실패와 관련된 정보를 필드로 보관하고 getter 제공
- 예외를 잡아 복구해야 하는 경우 특히 유용
```
public int getIndex() { return index; }
public int getMin() { return min; }
public int getMax() { return max; }
```
- 검사 예외(checked exception)에서 특히 효과적이지만, 비검사 예외라도 접근자 제공을 권장한다.

## 질문
- 예외 메세지를 자세하게 작성해야 하는 이유는 무엇인가요?
- 예외 상세 메세지에 어떤 정보를 포함해야 할까요?