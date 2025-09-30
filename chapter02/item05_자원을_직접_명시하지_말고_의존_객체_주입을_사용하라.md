# ITEM05.자원을 직접 명시하지 말고 의존 객체 주입을 사용하라

---
## 자원을 직접 명시하면 왜 문제가 될까? 
예를 들어, 맞춤법 검사기(SpellChecker)는 사전(Dictionary)에 의존한다.

#### 1. 정적 유틸리티 클래스
```
public class SpellChecker {
    private static final Dictionary dictionary = new KoreanDictionary();

    public static boolean isValid(String word) { ... }
}
```
#### 2. 싱글턴 방식
```
public class SpellChecker {
    private final Dictionary dictionary = new KoreanDictionary();
    private static final SpellChecker INSTANCE = new SpellChecker();

    private SpellChecker() {}
}

```
- 두 방식 모두 `SpellChecker` 는 오직 `KoreanDictionary` 만 쓸 수 있다.
  - 영어사전, 테스트용 가짜 사전(Mock) 을 사용할 수 없다.
- 즉, 유연하지 않고 테스트하기 어렵다는 한계가 있다.

#### dictionary 필드에서 final을 제거하고 setter를 추가하면?
- 런타임에 교체 가능하지만 → 어색하고, 오류 발생 가능성이 높다.
- 멀티스레드 환경에서는 레이스 컨디션(race condition) 발생할 수 있다.


## 해결 방법
해결 조건 :
- 여러 자원 인스턴스를 지원해야 하고
- 클라이언트가 원하는 자원을 사용해야 한다.
### 1) 생성자 주입
```
public class SpellChecker {
    private final Dictionary dictionary;

    public SpellChecker(Dictionary dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }

    public boolean isValid(String word) { ... }
}
```
```
SpellChecker korChecker = new SpellChecker(new KoreanDictionary());
SpellChecker engChecker = new SpellChecker(new EnglishDictionary());
```
- 이제 `Dictionary` 의 어떤 구현체든 클라이언트가 선택 가능하다.

### 2) 팩토리 주입
자원 자체가 아니라 **자원을 만들어주는 팩토리**를 넘길 수 있다.
```
public class Client {
    private final Supplier<Dictionary> dictionaryFactory;

    public Client(Supplier<Dictionary> dictionaryFactory) {
        this.dictionaryFactory = dictionaryFactory;
    }

    public void run() {
        Dictionary d = dictionaryFactory.get(); // 필요할 때마다 생성
    }
}
```
- 팩토리란?
  - 호출할 때마다 새로운 객체를 생성해주는 객체/함수
  - Supplier<T>는 자바 표준 함수형 인터페이스로, get() 호출 시마다 인스턴스를 만들어준다.
  - 즉, 의존성 생성 책임을 외부로 위임하는 방식

## 장점
- 유연성 : 다양한 자원 교체 가능
- 재사용성 : 같은 클래스가 여러 자원과 함께 쓰일 수 있음
- 테스트 용이성 : 가짜 객체를 주입 가능
- 불변성 보장 : final 필드로 고정 → 한 번 주입된 의존성은 변하지 않음
    - 불변 객체는 멀티스레드 환경에서도 동기화 없이 안전하게 공유 가능
    - SpellChecker 자체가 스레드 안전해짐 (단, 주입한 자원도 불변이어야 함)

## 단점
- 대규모 프로젝트: 수십~수백 개의 의존성을 관리하기 힘듦 
    → DI 프레임워크(Spring, Guice, Dagger) 필요

## 결론
- 클래스가 자원에 의존하고 있고, 자원이 클래스 동작에 영향을 준다면 정적 유틸리티 클래스나 싱글턴을 이용하지 않는 게 좋다
- 자원들을 클래스가 직접 만들게 해서도 안된다
- 필요한 자원(또는 그 자원을 만들어주는 팩터리)을 생성자(또는 정적 팩터리나 빌더)에 넘겨주면 클래스의 유연함, 재사용성, 테스트 용이성을 개선해준다