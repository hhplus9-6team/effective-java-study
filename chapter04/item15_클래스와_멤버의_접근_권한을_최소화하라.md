# item15. 클래스와 멤버의 접근 권한을 최소화하라

## 1. 캡슐화

### 1) 캡슐화(정보은닉)란?
내부 구현을 외부에서 직접 접근하지 못하게 숨기고, 명확히 정의된 인터페이스(API) 로만 접근하도록 제한하는 원칙

### 2) 장점

- 시스템 개발 속도를 높임
  - 여러 컴포넌트를 병렬로 개발 가능
- 시스템 관리 비용을 낮춤
  - 각 컴포넌트를 더 빨리 파악하여 디버깅 가능
  - 다른 컴포넌트로 교체하는 부담이 적음
- 성능 최적화에 도움을 줌
  - 다른 컴포넌트에 영향을 주지 않고 해당 컴포넌트만 최적화 가능
- 소프트웨어 재사용성 높임
  - 외부에 거의 의존하지 않고 독자적으로 동작할 수 있는 컴포넌트라면 낯선 환경에서도 유용하게 사용 가능
- 큰 시스템 제작 난이도 낮춤
  - 시스템 전체가 완성되지 않은 상황에서 개별 컴포넌트 동작 검증 가능

## 2. 접근 제어 메커니즘

### 1) 접근 제어 메커니즘이란?

- 자바에서 클래스, 메서드, 필드 등 멤버에 접근할 수 있는 범위를 통제하는 시스템
- 각 요소의 접근성은 그 요소가 **선언된 위치**와 **접근 제한자**로 정해짐

### 2) 선언 위치

#### (1) 최상위(top-level) 클래스

```java
// 같은 파일 내에선 이렇게 가능
class PackagePrivateClass {} // default
public class PublicClass {}   // public

// ❌ 아래는 불가능 (컴파일 에러)
private class PrivateClass {}
protected class ProtectedClass {}
```

- 최상위 클래스(파일 맨 바깥)는 `public` 혹은 `default`만 가능
- 선언 위치가 최상위이기 때문에 `private` 나 `protected`는 쓸 수 없음

#### (2) 중첩(nested) 클래스나 멤버

```java
public class Outer {
    private int x;              // 외부 클래스에서 접근 불가
    protected void print() {}   // 하위 클래스만 접근 가능
    class Inner {}              // default → 같은 패키지 내에서만 접근 가능
}
```

- 클래스 내부에 선언된 필드, 메서드, 내부 클래스는 `private`, `protected`, `public`, `default` 모두 가능

### 3) 접근 제한자

- **private** : 멤버를 선언한 톱레벨 클래스에서만 접근 가능
- **default (package-private)** : 멤버가 소속된 패키지 안의 모든 클래스에서 접근 가능
  - 접근 제한자 명시하지 않았을 때 적용되는 패키지 접근 수준
- **protected** : package-priate의 접근 범위를 포함하며, 이 멤버를 선언한 클래스의 하위 클래스에서도 접근 가능
- **public** : 모든 곳에서 접근 가능

### 4) 유의사항

#### (1) 기본 원칙 : 접근성은 가능한 한 최소한으로 공개하기

- 모든 클래스와 멤버의 접근성을 가능한 한 좁혀야 합니다.
- 패키지 외부에서 쓸 이유가 없다면 `package-private`으로 선언해야 합니다.
- 한 클래스에서만 사용하는 `package-private` 톱레벨 클래스나 인터페이스는 이를 사용하는 클래스 안에 private static으로 중첩시켜 봅시다.
    ```java
    public class Service {
        private static class Helper { ... } // Service 내부 전용
    }
    ```
  
#### (2) Serializable 구현 주의

- Serializable을 구현한 클래스에서는 그 필드들도 의도치 않게 공개 API가 될 수 있습니다.
    - 직렬화를 한다는 것 자체가 클래스의 필드 구성 자체가 외부에 노출되어 공개가 되는 것이므로, 가능하면 Serializable 구현을 피하라는 의미
- 직렬화가 꼭 필요하다면 `serialVersionUID`, `transient`, `writeObject` / `readObject` 로 제어해야 합니다.

#### (3) 상속 시 접근 수준

- `protected`는 내부 동작 방식을 API 문서에 적어 사용자에게 공개해야 할 수 있으므로, 상속 설계가 명확할 때만 사용해야 합니다.
- 상위 클래스의 메서드를 재정의할 때는 그 접근 수준을 상위 클래스에서보다 좁게 설정할 수 없습니다.
    - 리스코프 치환 원칙 : 상위 클래스의 인스턴스는 하위 클래스의 인스턴스로 대체해 사용할 수 있어야 합니다.

#### (4) 테스트 시 접근 완화 금지

- 테스트만을 위해 클래스, 인터페이스, 멤버를 공개 API로 만들어서는 안 됩니다.
- 패키지 구조를 테스트와 공유하면 클래스 접근이 가능합니다. 

#### (5) public 가변 필드 금지

- public 클래스의 인스턴스 필드는 되도록 public이 아니어야 합니다.
  - public 가변 필드를 갖는 클래스는 일반적으로 스레드 안전하지 않습니다.
- 상수라면 public static final 필드로 공개해도 좋지만, public static final 배열 필드나 배열 필드를 반환하는 접근자 메서드를 제공해서는 안 됩니다.
  - 해결방안 1: public 배열을 private 로 반들고 public 불변 리스트 추가
    ```java
    private static final Thing[] PRIVATE_VALUES = {...};
    public static final List<Thing> VALUES = Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
    ```
  - 해결방안 2: 배열을 private으로 반들고 그 복사본을 반환하는 public 메서드 추가
    ```java
    private static final Thing[] PRIVATE_VALUES = {...};
    public static final Thing[] values() {
        return PRIVATE_VALUES.clone();
    }
    ```

## 3. 모듈 시스템 (JAVA 9+)

### 1) 등장 배경

자바 8까지는 패키지 수준의 접근 제어가 유일했습니다.
패키지 외부에서 접근하지 못하게 하려면 `package-private` 으로 선언하면 됐지만, JAR 단위로 배포되면 결국 모든 코드가 외부에서 접근 가능한 상태가 됐습니다.

#### 내부 구현 (라이브러리)
```java
// mylib.jar
package com.company.internal;

class InternalHelper {
    static void doSomething() { ... }
}
```
#### 외부 애플리케이션
```java
// app.jar
package com.company.internal;

public class Exploit {
    static { InternalHelper.doSomething(); } // ⚠️ 접근 가능!
}
```

즉, 내부 구현 클래스도 다른 개발자가 import 해서 쓸 수 있었습니다. 
이는 캐슐화 실패이며, API 안정성과 보안성 모두를 해쳤습니다.

이 문제를 해결하기 위해 Java 9 (2017) 에서 새로운 접근 제어 체계인 **모듈 시스템 (JPMS, Java Platform Modeule System)** 이 도입되었습니다.

### 2) 핵심 개념

- 모듈(module) : 여러 패키지를 묶은 배포 단위
- module-info.java : 모듈의 공개 API, 의존 관계를 선언하는 파일
- exports : 외부에 공개할 패키지 지정
- requires : 이 모듈이 의존하는 다른 모듈 명시
- opens : 리플렉션 전용 접근 허용 (Spring, Jackson 등의 프레임워크용)

### 3) 구조 예시

#### 디렉토리 구조
```
src/
 └─ com.myapp.core/
     ├─ module-info.java
     ├─ com/myapp/api/
     │    └─ UserService.java
     └─ com/myapp/internal/
          └─ UserRepository.java
```

#### module-info.java
```java
module com.myapp.core {
    exports com.myapp.api; // 외부에 공개할 패키지
}
```

- `com.myapp.api`는 외부 접근 가능
- `com.myapp.internal`은 export 되지 않았으므로 같은 모듈 내부에서만 접근 가능

즉, 패키지보다 상위 수준의 정보 은닉이 가능해졌습니다.

#### opens : 리플렉션 허용 제어

```java
module com.myapp.core {
    exports com.myapp.api;
    opens com.myapp.internal to spring.core, jackson.databind;
}
```

- `com.myapp.internal`은 여전히 일반 코드에서는 접근 불가
- `spring.core`, `jackson.databind` 모듈은 리플렉션 접근 가능

모듈 시스템은 기본적으로 리플렉션을 차단하지만, 스프링, 하이버네이트, 잭슨 같은 프레임워크는 리플렉션을 사용해야 합니다.

이 때, `opens` 키워드를 사용하면 특정 패키지만 리플렉션 전용으로 열 수 있습니다.

#### requires : 모듈 의존성

```java
module com.myapp.web {
    requires com.myapp.core;
    requires java.sql;
}
```

- `com.myapp.web`은 `com.myapp.core` 모듈과 `java.sql` 모듈에 의존
- 이 선언이 없으면 해당 모듈의 클래스를 import 불가

다른 모듈의 기능을 사용하려면 `requires` 명시가 필요합니다.

### 4) 클래스패스 vs 모듈 경로

실행 방식	| 설명      | 결과
-|---------|-
`--module-path mods` | 모듈로 인식  | `exports` 기준 접근 제어 적용
`-classpath libs/*` | 일반 JAR로 인식 | `module-info.java` 완전히 무시, 모든 패키지 접근 가능

모듈의 JAR 파일을 클래스패스에 두면, 그 모듈은 모듈이 아닌 평범한 JAR 처럼 동작합니다.

모듈 시스템의 효과를 얻으려면 반드시 `module-path`로 실행해야 합니다.

그렇지 않으면 내부 구현이 그대로 노출되어 모듈화의 의미가 사라집니다.

### 5) JDK 자체의 모듈화

Java 9에서 JDK 자체가 완전히 모듈화되었습니다.

```
java.base
java.logging
java.sql
java.desktop
java.xml
...
```

예전에는 모든 표준 클래스가 `rt.jar` 안에 들어 있었지만, 지금은 수십 개의 모듈로 분리되어 있습니다.

즉, 모듈 시스템을 실제로 가장 적극적으로 적용한 사례가 바로 JDK 입니다.

하지만, 대부분의 프로젝트는 여전히 클래스패스 기반 빌드를 사용하고 있습니다.

Gradle, Maven 등의 빌드 시스템에서 모듈 지원이 제한적이고, Spring, Hibernate 등 리플렉션 의존 프레임워크와 충돌이 발생합니다.

따라서 실무에서는 여전히 JAR 수준의 모듈 관리로 충분하다고 여겨져서, JDK 외에도 모듈 개념이 널리 받아들여질지는 아직 이릅니다.


#### 6) 멀티모듈 프로젝트?

```
root/
 ├─ build.gradle
 ├─ settings.gradle
 ├─ core/
 │   └─ build.gradle
 ├─ web/
 │   └─ build.gradle
 └─ common/
     └─ build.gradle
```

- Gradle의 모듈은 단순히 **빌드 단위(subproject)** 를 의미합니다.
- 멀티 모듈 프로젝트는 빌드 단위를 여러 개로 나눈다는 의미입니다. 하나의 리포지토리 안에 여러 프로젝트를 관리하는 구조일 뿐입니다.
- Gradle의 멀티모듈 프로젝트는 빌드 단위 구분일 뿐, 언어적 모듈화(JPMS)와는 전혀 별개의 개념입니다.
- Java의 모듈 시스템은 Java 9에서 도입된 언어 수준의 캡슐화 체계로, 패키지를 넘어 모듈 단위로 접근 권한을 통제할 수 있게 하는 것입니다.
- Gradle 멀티모듈 프로젝트 안에서도 각 서브모듈마다 module-info.java를 추가해 JPMS 모듈로 구성할 수는 있습니다.
  하지만 대부분의 프로젝트는 그렇게 하지 않습니다. 왜냐하면 Spring, Hibernate 같은 프레임워크들이 아직 완전히 호환되지 않기 때문입니다.

---

### 질문

Q. 정보 은닉(캡슐화)의 장점은 어떤 것이 있을까요?

Q. 다음 중 올바른 접근 제한자 설명으로 가장 적절하지 않은 것은?
1) private : 같은 클래스 안에서만 접근 가능
2) default(package-private) : 같은 패키지 내 모든 클래스에서 접근 가능
3) protected : 하위 클래스와 다른 패키지에서도 접근 가능
4) public : 모든 패키지, 모든 클래스에서 접근 가능

---

### 참고 자료

- Joshua Bloch, Effective Java, 3rd Edition, Item 15