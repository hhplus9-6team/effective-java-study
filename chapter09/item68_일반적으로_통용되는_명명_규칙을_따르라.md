# item68. 일반적으로 통용되는 명명 규칙을 따르라

자바 플랫폼은 명명 규칙이 잘 정립되어 있으며, 그중 많은 것이 자바 언어 명세에 기술되어 있다.
명명 규칙은 크게 **철자**와 **문법** 두 범주로 나뉜다.
이 규칙을 어긴 API는 유지보수가 어렵고 사용하기 어렵다.

## 1. 철자 규칙
### 1.1 패키지/모듈 이름 
- 점(.)으로 구분된 계층 구조
- 모든 요소는 소문자
- 외부 공개 패키지는 인터넷 도메인 역순
  - com.google, org.eff, edu.cmu
- 표준 라이브러리는 예외적으로
  - java.*, javax.*

#### 패키지 하위 요소 규칙
- 의미가 통하는 짧은 단어(보통 8자 이하) 사용
  - utilities ❌ → util ✅
- 여러 단어라면 약어 허용
  - abstract window toolkit → awt
- 기능이 많을수록 하위 패키지로 세분화
  - java.util.concurrent.atomic

### 1.2 클래스/ 인터페이스 이름
- 하나 이상의 단어로 이뤄지며, 각 단어는 대문자로 시작한다.
  - List, FutureTask
- 널리 통용되는 약어 외에는 축약 금지
  - Cnt, Mgr ❌
- 약어 대소문자 규칙은 일관성만 유지하면 됨
  - HttpServer / HTTPServer 중 하나로 통일

### 1.3 메서드와 필드 이름
- 첫글자만 소문자로 쓴다는 점만 빼면 클래스 명명 규칙과 같다.
- 첫 단어가 약자라면 단어 전체가 소문자여야 한다.

#### 상수 필드 (static final)
- 모두 대문자로 쓰며 단어 사이는 밑줄로 구분한다.
  - VALUES, NEGATIVE_INFINITY
- 언더스코어를 허용하는 유일한 경우

#### 지역 변수
- 약어를 써도 좋다. 약어를 써도 그 변수가 사용되는 문맥에서 의미를 쉽게 유추할 수 있기 때문이다.
- 입력 매개변수도 지역변수 중 하나이지만 메서드 설명 문서에서까지 등장하는 만큼 일반 지역변수보다는 신겨써야 한다.

### 1.4 타입 매개변수
- 관례적으로 한 문자
  - T : Type
  - E : Element
  - K, V : Key, Value
  - X : Exception
  - R : Return type
- 여러 개일 경우
  - T, U, V 또는 T1, T2

## 2. 문법 규칙

### 2.1 클래스

#### 인스턴스를 생성할 수 있는 클래스
- 단수 명사
- Thread, PriorityQueue

#### 인스턴스를 생성할 수 없는 클래스 (유틸리티 클래스)
- 복수형 명사
- Collections, Collectors

### 2.2 인터페이스
- 클래스와 동일한 명사 형태 또는 -able, -ible로 끝나는 형용사
- Runnable, Iterable, Accessible

### 2.3 애너테이션
- 명확한 규칙은 없으나 의미 중심
  - 명사: Singleton
  - 동사: Inject
  - 형용사/구문: BindingAnnotation

### 2.4 메서드
#### 동작 수행
- 동사 / 동사구
  - append, drawImage

#### boolean 반환
- is, has로 시작
  - isEmpty
  - hasChildren

#### 속성 반환 (boolean 제외)
- 명사 / 명사구 또는 get
  - size()
  - hashCode()
  - getTime()

#### 타입 변환
- toType
  - toString(), toArray()

#### 다른 뷰 제공
- asType
  - asList

#### 기본 타입 반환
- typeValue
  - intValue, longValue

#### 정적 팩터리 메서드
- of
- from
- valueOf
- instance
- getInstance

### 2.5 필드 이름
- boolean 필드
  - 접근자에서 is를 제거한 형태
  - initialized
  - composite
- 그 외 필드
  - 명사 또는 명사구
  - 축약 최소화

## 질문
- 왜 Collections와 Collectors 같은 클래스는 복수형 이름을 사용하고, List나 Map은 단수형 이름을 사용할까요?