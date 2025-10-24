# Item 25. 톱레벨 클래스는 한 파일에 하나만 담아라

## 톱레벨 클래스에서 다른 톱레벨 클래스 2개를 참조하는 경우

```java
// Main.java
public class Main {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }
}
```
```java
// Utensil.java
class Utensil {
    static final String NAME = "pan";
}

class Dessert {
    static final String NAME = "cake";
}
```
```bash
javac Main.java Utensil.java
> pancake
```

Main 클래스를 실행하면 `pancake`가 출력된다.

### 중복 정의 문제

이 상태에서 똑같은 두 클래스를 담은 `Dessert.java`가 추가되면 문제가 발생한다.
```java
// Dessert.java
class Utensil {
    static final String NAME = "pot";
}

class Dessert {
    static final String NAME = "pie";
}
```
```bash
javac Main.java Dessert.java
> Duplicate class found in the file '~/src/main/java/Dessert.java'
```

컴파일 오류가 발생하고 클래스가 중복 정의되었다고 알려준다.

### 컴파일 순서

컴파일러는 다음 순서로 동작한다:

1. 가장 먼저 `Main.java`를 컴파일
2. `Utensil` 참조를 만나면 `Utensil.java` 파일을 살펴봄
3. `Utensil.java`에서 `Utensil`, `Dessert` 클래스를 모두 찾아냄
4. 두 번째로 `Dessert.java`를 처리하려 할 때 같은 클래스가 이미 정의되어 있음을 발견

### 예측 불가능한 동작

컴파일러에 어느 소스 파일을 먼저 전달하느냐에 따라 동작이 달라진다.
```bash
javac Main.java
> pancake

javac Main.java Utensil.java
> pancake

javac Dessert.java Main.java
> potpie
```

컴파일 순서에 따라 프로그램의 동작이 달라지면 안 된다.

## 해결책

### 1. 톱레벨 클래스를 서로 다른 소스 파일로 분리 
```java
// Utensil.java
class Utensil {
    static final String NAME = "pan";
}
```
```java
// Dessert.java
class Dessert {
    static final String NAME = "cake";
}
```

### 2. 정적 멤버 클래스 사용 
```java
// Test.java
class Test {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }

    private static class Utensil {
        static final String NAME = "pan";
    }

    private static class Dessert {
        static final String NAME = "cake";
    }
}
```
Q. 같은 파일에 여러 톱레벨 클래스를 두면 어떤 문제가 발생할 수 있을까?
