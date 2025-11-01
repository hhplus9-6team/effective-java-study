# item36. 비트 필드 대신 EnumSet을 사용하라

## 비트 필드란,
- 여러 상수를 하나의 집합으로 표현하기 위해 각 상수에 서로 다른 2의 거듭제곱 값을 부여하는 정수 기반 열거 패턴이다.
- 여러 옵션을 비트별 OR(|) 연산으로 결합한다.
```
public class Text {
    public static final int STYLE_BOLD = 1 << 0;
    public static final int STYLE_ITALIC = 1 << 1;
    public static final int STYLE_UNDERLINE = 1 << 2;
    public static final int STYLE_STRIKETHROUGH = 1 << 3;
    
    //매개변수 styles 는 0개 이상의 STYLE_ 상수를 비트별 OR 한 값이다.
    public void applyStyles(int styles) { .. }
}
```
```
text.applyStyles(STYLE_BOLD | STYLE_ITALIC);
```

### 단점
- 읽기 어렵다.
  - 5 같은 값이 어떤 상수들의 조합인지 직관적으로 알 수 없다.
- 순회 불편
  - 포함된 항목을 확인하려면 매번 비트 마스크 연산을 해야 한다.
  - ```if ((styles & STYLE_BOLD) != 0) {...}```
- 확장이 어렵다.
  - 32개를 넘으면 int → long 으로 바꿔야 하고, 관련 코드/DB/프로토콜까지 수정 필요하다.


## EnumSet 클래스
- `java.util` 패키지
- Set 인터페이스를 완전하게 구현하여 타입 안전하다.
- 다른 어떤 Set 구현체와도 함께 사용할 수 있다.
- 내부적으로는 비트 벡터(bit vector) 로 구현되어, 원소가 64개 이하라면 Long 한 개로 표현되어 비트 필드만큼 빠르다.
- 대량 연산(removeAll, retainAll)도 비트 연산으로 최적화됨.
```
public class Test {
    public enum Style { BOLD, ITALIC, UNDERLINE, STRIKETHROUGH }
    
    //어떤 Set 을 넘겨도 좋으나 EnumSet 이 가장 좋다.
    public void applyStyles(Set<Style> styles) { ... }
```
```
text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
```

