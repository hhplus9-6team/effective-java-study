# item56. 공개된 API 요소에는 항상 문서화 주석을 작성하라

### 1. 자바독(Javadoc)의 역할
- 소스코드에 작성된 문서화 주석을 기반으로 API 문서를 생성한다.
- 문서화 주석이 없다면 API 선언만 나열될 뿐이며, API 사용자에게 필요한 설명을 제공하지 못한다.

### 2. 문서화 주석을 작성해야 하는 대상
#### 반드시 작성해야 하는 공개 API
- 공개 클래스, 인터페이스
- 공개 메서드, 생성자
- 공개 필드

#### 기본 생성자 사용 금지
기본 생성자는 문서화 주석을 달 수 없기 때문에 공개 클래스는 기본 생성자가 생기지 않도록 반드시 명시적 생성자를 선언해야 한다.

#### 공개되지 않은 코드에도 주석 작성 권장
유지보수를 고려하면 `package-private`, `protected`, `private` 멤버에도 문서화 주석이 매우 유용하다.

### 3. 메서드 문서화 주석의 핵심: 클라이언트와의 계약을 기술한다.
#### 메서드가 무엇을 하는가?에 집중
- (상속을 위한 메서드가 아니라면) 내부 구현이나 알고리즘이 아니라 행동을 설명해야 한다.

#### 계약 요소
1) 전제 조건 (preconditions)
   - 메서드를 호출하기 위해 충족되어야 할 조건
   - 관례
     - `@throws` 태그로 비검사 예외를 선언하여 조건을 기술
     - `@param` 태그로 특정 매개변수의 유효 조건을 직접 명시

2) 사후 조건
   - 메서드 실행 성공 이후 반드시 보장되는 상태
3) 부작용
   - 사후 조건으로 보이지 않지만 시스템 상태를 변화시키는 경우
   - 예시) 백그라운드 스레드를 시작함, 캐시를 변경함 등
   - 반드시 문서화해야 한다.

### 4. 표준 태그 활용 방식
```
/**
 * Returns the element at the specified position in this list.
 *
 * @param index index of the element to return
 * @return the element at the specified position in this list
 * @throws IndexOutOfBoundsException if index < 0 || index >= size()
 */
E get(int index);
```
메서드의 계약을 완전히 기술하기 위해서는 아래 3가지를 모두 문서화해야 한다.
#### 1. 모든 매개변수에 `@param`
- 각 매개변수가 무엇을 의미하는지 명사구로 설명
- 예시 ) "the initial capacity", "the index of the element to remove"
#### 2. 반환 타입이 void 가 아니면 `@return`
- 반환값이 무엇인지 명사구로 설명
- 예시 ) "the number of elements", "a new immutable copy of the list"
#### 3. 발생 가능한 모든 예외에 `@throws`
- 어떤 상황에서 어떤 예외가 발생할 수 있는지를 기술
- if + 조건절 형태로 설명하는 것이 일반적

```
@implSpec
This implemntation returns {@code this.size() == 0}.

@return true if this collections is empty
public boolean isEmpty() { ... }
```
#### 4.`{@code ...}` 태그
- 코드 폰트로 렌더링
- 내부의 HTML/문서화 주석 태그는 해석되지 않음.
- 예시 ) "{@code this.size() == 0}"
#### 5. `implSpec` 태그
- 상속을 위한 메서드의 동작을 설명(하위 클래스와의 계약)
- 하위 클래스 객체가 super.foo()를 호출했을 때 어떤 동작이 보장되는지 명시
#### 6. `{@literal}` 태그
- API 설명에 <, >, & 등의 HTML 메타 문자를 포함시키려면 특별한 처리를 해줘야 한다.
- HTML 마크업이나 자바독 태그를 무시하게 해준다.
- 코드 폰트로 렌더링하지 않는다.
```
* A geometric series coverges if {@literal |r| < 1}.
```
       - 인스턴스 메서드의 문서화 주석에 쓰인 this 는, 호출된 메서드가 자리하는 객체를 카리킨다.


       - 관례상 `@param`, `@return`, `@throws` 태그의 설명에는 마침표를 붙이지 않는다.

       - 드물게는 명사구 대신 산술 표현식을 작성하기도 한다.
         - ex) BigInteger : if 로 시작해 해다 예외를 던지는 조건을 설명하는 절이 뒤따른다.

### 5. 요약 설명
- 문서화 주석의 첫 문장이 API 문서에서 요약 설명으로 사용됨.
- 반드시 해당 기능을 고유하게 설명해야 한다.
- 동일 클래스 내에서 같은 요약 설명을 재사용하면 안 됨 (오버로드 메서드도 예외 아님)

#### 문장 구분 규칙
- “마침표 + 공백 + 소문자가 아닌 문자” 조합이 나타날 때까지가 요약 문장
- 이메일, 약어 등 의도치 않은 마침표는 {@literal}로 묶어야 함

#### 요약 설명의 문체
- 메서드/생성자 → 동사구로 시작
  - “Returns …”
  - “Creates …”
- 클래스/인터페이스/필드 → 명사구
  - “Immutable list implementation”
  - “Count of elements …”

### 6. 검색 기능
#### `{@index}` 태그
- API 문서 내 색인에 항목 추가
```
* This method complies with the {@index IEEE 754} standard.
```

### 7. 제너렉, 열거 타입, 애너테이션은 주의
#### 제너릭 타입/메서드
- 제너릭 타입이나 제너릭 메서드를 문서화할 때에는 모든 타입 매개변수에 @param <T> 형태로 설명을 반드시 추가해야 한다.
```
/**
 * A mapping between keys and values.
 *
 * @param <K> the type of keys maintained by this map
 * @param <V> the type of mapped values
 */
public interface Map<K, V> {

    /**
     * Returns the value to which the specified key is mapped.
     *
     * @param key the key whose associated value is to be returned
     * @return the value mapped to the specified key, or null if none
     */
    V get(Object key);
}
```

#### 열거 타입
- 각 상수에도 문서화 주석을 달아야 한다.
- 열거 타입 자체 및 모든 public 메서드도 문서화 대상이다.
- 설명이 짧을 경우 한 문장으로 간단히 작성해도 된다.
```
/**
 * Represents sections of a standard orchestra.
 */
public enum OrchestraSection {

    /** Woodwind instruments such as flute and clarinet. */
    WOODWIND,

    /** Brass instruments including trumpet and trombone. */
    BRASS,

    /** Percussion instruments such as drums and cymbals. */
    PERCUSSION,

    /** String instruments like violin and cello. */
    STRING;

    /**
     * Returns true if this section uses primarily acoustic instruments.
     *
     * @return true if this section uses acoustic instruments
     */
    public boolean isAcoustic() {
        return this != PERCUSSION;
    }
}
```

#### 애터네이션 타입
- 애너테이션 타입의 요약 설명은 동사구로 작성
  - (예: “Indicates that …”, “Marks a method …”)
- 각 멤버(element)에는 명사구로 주석 작성
  - (예: “the error message format”, “whether logging is enabled”)
- 모든 멤버에 반드시 문서화 주석을 추가해야 한다.
```import java.lang.annotation.*;

/**
 * Indicates that the annotated method provides custom exception text.
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionText {

    /**
     * The text to be used when formatting the exception message.
     */
    String value();

    /**
     * Whether the exception should include stack trace information.
     */
    boolean includeStackTrace() default true;
}
```

### 8. 그외 주의해야 할점
#### 패키지 문서화
- package-info.java 파일에 작성
- 패키지 레벨의 목적, 범위, 사용 예시 등을 설명하는 것이 좋음.

#### 스레드 안정성 명시
- 클래스나 메서드가 스레드 안정한지 여부를 반드시 API 문서에 포함해야 한다.

#### 직렬화 가능성
- Serializable 여부, 직렬화 형태, 직렬화 규약이 변하지 않는다는 점 등을 API 문서에 명시해야 한다.

#### 메서드 주석 상속 (@inheritDoc)
- 하위 클래스가 문서화를 생략하더라도 상위 클래스/인터페이스의 주석을 자동으로 상속할 수 있다.
- 그러나 필요시 명확한 문법적 기술을 병행하는 것이 좋다.

#### 복잡한 상호작용 API라면 별도 문서 필요
- 시스템 아키텍처 문서, 디자인 문서 등
- JavaDoc에서 해당 문서를 링크로 연결하는 방식 추천