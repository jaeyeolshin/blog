---
title: Java 8 - Interface의 변화
date: 2017-04-19 16:32
categories:
- Java
tags:
- java
- interface
---

요즘 [카이 호스트만의 코어 자바8](http://www.yes24.com/24/goods/23449538?scode=032&OzSrank=1)로 Java 8을 공부하고 있다. 작년 초에 개발자로 전향하여 Java 8의 stream 같은 것을 ~~매우 신기해하며~~ 어깨너머로 배워 사용했었는데 알고보니 Java 8이 처음 출시된 것은 2014년 3월 18일이었다.([Java 릴리스 페이지 참고](https://www.java.com/ko/download/faq/release_dates.xml)) 지금은 Java 9가 나오려고 몸을 들썩이는 중이다. Java 8을 제대로 공부도 해보지 않았는데 벌써 Java 9가 나온다고 하니 젊은 나이에 뒤처지는 것 같은 기분이 들었다. 이런 까닭에 황급히 서점에서 책을 사서 공부하는 중이다. 원저의 제목 [Core Java for the Impatient](https://www.amazon.com/Core-Java-Impatient-Cay-Horstmann/dp/0321996321)가 내 상황에 잘 들어맞는 것 같다. 더 늦기 전에 Java 8을 공부하면서 중요하다고 생각하는 것들을 정리해보려고 한다.<!-- more -->

# 인터페이스

객체 지향 프로그래밍에서 인터페이스는 기능의 생김새만 나타낸다. 인터페이스는 어떤 기능에 대한 추상이며, 실제 구현은 그 인터페이스를 구현하는 클래스에게 맡긴다. 해당 인터페이스를 사용하는 입장에서는 실제 클래스가 어떻게 구현되어 있는지 몰라도 인터페이스의 생김새에 따라 함수를 호출하기만 하면 된다. 마치 복잡한 시스템의 UI(유저 인터페이스)와 같다. 구글 검색 엔진은 복잡한 시스템이지만 사용자에게 보여주는 건 질의어를 입력하는 텍스트 박스밖에 없다.
![구글 검색 엔진](/images/java8-changes-in-interface/google.png)
<p style="text-align:center;color:#808080;">검색 엔진의 구현을 몰라도 검색을 할 수 있다(이미지 출처: <a href="https://www.google.com">구글</a>)</p>
추상화가 잘되어 있다는 것은 (구글 검색 엔진처럼)객체의 필요한 기능만 드러내고, 복잡하고 굳이 드러내지 않아도 되는 내용들은 숨겼다는 것을 의미한다. 이전에는 이러한 인터페이스의 추상성을 철저히 지켰기 때문에 인터페이스가 어떤 상태(인스턴스 변수)나 구현된 메서드를 갖는 것이 불가능했다. 하지만 Java 8부터 인터페이스가 조금 더 유연하게 바뀌었다.

# 정적 메서드

기술적으로 Java에서 인터페이스에 정적 메서드를 추가하지 못할 이유는 없었다. 정적 메서드는 어차피 인스턴스와 관계가 없기 때문이다. 다만 정적 메서드도 구현된 메서드라는 점에서 인터페이스의 추상성을 해친다는 것이 문제였다. Java 8에서는 그러한 제약이 없어졌고, 인터페이스에 정적 메서드를 추가할 수 있게 되었다.(사실 이전에도 인터페이스에 정적 필드는 정의할 수 있었기 때문에 정적 메서드가 Java 8에 와서야 추가된 것은 조금 의아하다.) 기존의 제약을 깨고 정적 메서드를 추가한 것은 개발 편의성을 높이려는 시도로 보인다. Java 8 이전의 표준 라이브러리에서는 인터페이스와 관련된 정적 메서드들을 동반 클래스(companion class)에서 제공했다. 대표적인 예로 [Collection](https://docs.oracle.com/javase/8/docs/api/java/util/Collection.html) 인터페이스와 [Collections](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html) 동반 클래스가 있다.

```java
// 인터페이스와 동반 클래스의 예.
Collection<String> empty = Collections.emptyList();
```

이제는 인터페이스에 바로 정적 메서드를 추가할 수 있기 때문에 동반 클래스를 따로 정의하지 않아도 된다. Java 8에 추가된 [Stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html) 인터페이스는 유용한 정적 메서드들을 제공한다.

```java
Stream<String> chosunKings     = Stream.of("태조", "정종", "태종", "세종", "문종", "단종", "세조", ...);
Stream<String> southKoreaKings = Stream.empty();
```

# 기본 메서드(default method)

Java 8에서는 인터페이스에 기본 구현을 정의할 수 있게되었다. 기본 구현이 제공되는 메서드는 구현 클래스에서 구현하지 않아도 컴파일이 가능하다. 기본 메서드는 기존의 인터페이스에 메서드를 추가해야하는 경우에 아주 유용하다. 인터페이스가 변경되는 일이 없도록 프로그램을 잘 작성하는게 좋겠지만 변경이 불가피한 상황이 생길 수도 있다. 인터페이스에 메서드를 추가하면 해당 인터페이스를 구현하는 모든 클래스에서 추가된 메서드를 구현해야하기 때문에 문제가 생긴다. 구현 클래스가 9개라면 인터페이스까지 10개의 파일을 수정해야 한다. 하지만 추가되는 메서드의 구현이 대부분 동일하다면 인터페이스에 기본적인 메서드 구현을 정의하고 유별난 클래스만 수정해주면 된다. 연료 유형을 포함하는 `Car` 인터페이스를 예로 들어보자.

```java
public interface Car {
    String fuelType();
}
```

연료 유형에 따른 구현 클래스들도 있다.
```java
public class DieselCar implements Car {
    @Override
    public String fuelType() {
        return "DIESEL";
    }
}
```
```java
public class GasolineCar implements Car {
    @Override
    public String fuelType() {
        return "GASOLINE";
    }
}
```

자동 주행 차량에 발빠르게 대응하기 위해서 `Car` 인터페이스에 자동 주행 차량 여부를 확인할 수 있는 메서드가 추가되어야한다고 생각해보자. `Car`는 아래와 같이 변경되어야 한다.

```java
public interface Car {
    String fuelType();
    boolean autodrive();
}
```

이 경우에 `autodrive()` 메서드는 기본 구현을 제공하지 않으므로 `DieselCar`, `GasolineCar`에서 구현해줘야 한다. 하지만 기존 차량들은 자율 주행이 안될 것이기 때문에 아래와 같이 기본 구현을 제공할 수 있다.

```java
public interface Car {
    String fuelType();
    default boolean autodrive() {
        return false;
    }
}
```

`autodrive()`는 `FutureCar`와 같은 유별난 클래스에서만 따로 구현해주면 된다.
```java
public class FutureCar implements Car {
    @Override
    public String fuelType() {
        return "SOLAR";
    }

    @Override
    public boolean autodrive() {
        return true;
    }
}
```

인터페이스의 기본 메서드는 클래스의 계층을 좀 더 단순하게 만들어준다는 장점도 있다. Java 2부터 있어왔던 [AbstractCollection](https://docs.oracle.com/javase/8/docs/api/java/util/AbstractCollection.html)은 [Collection](https://docs.oracle.com/javase/8/docs/api/java/util/Collection.html) 구현 클래스들의 공통 기능을 제공한다. Java 8 이전에는 구현 클래스들의 공통 기능들을 묶기 위해 인터페이스와 구현 클래스 사이에 추상 클래스를 정의하는 것이 일반적이었다. 하지만 Java 8에 와서는 더 이상 추상 클래스를 추가할 필요 없이 기본 메서드를 정의할 수 있게 되었다. 이런 변화로 인터페이스와 추상 클래스의 경계가 모호해졌다는 느낌이 들지만 여전히 인스턴스 변수의 유무 차이는 존재한다.

# 기본 메서드의 충돌 해결하기

Java에서 하나의 클래스는 여러 인터페이스를 구현할 수 있다. Java 8 이전에는 여러 인터페이스가 같은 메서드를 갖더라도 어차피 구현은 클래스에서만 제공했기 때문에 문제가 되지 않았다. 하지만 Java 8에서 인터페이스들이 각각 동일한 메서드의 기본 구현을 제공하고, 클래스에서 충돌이 발생하는 메서드를 명시적으로 오버라이드 하지 않으면 컴파일러가 어떤 기본 메서드를 사용해야할 지 선택할 수 없기 때문에 문제가 발생한다. 책에 나와있는 예시를 살펴보자.

```java
public interface Person {
    String getName();
    default int getId() { return 0; }
}
```
```java
public interface Identified {
    default int getId() { return Math.abs(hashCode()); }
}
```
```java
public class Employee implements Person, Identified {
    ...
}
```

`Person`과 `Identified` 인터페이스는 `getId()` 기본 메서드를 정의하고 있고, `Employee` 클래스는 두 인터페이스를 구현한다. `Employee` 클래스에서 `getId()`를 오버라이드 하지 않으면 **Employee inherits unrelated defaults for getId() from types Person and Identified** 라는 컴파일 에러가 발생한다.

한 쪽에서 기본 메서드를 구현하지 않으면 문제가 해결될까? `Identified`를 아래와 같이 기본 메서드 구현을 하지 않도록 바꾸고 컴파일 해보자.

```java
public interface Identified {
    int getId();
}
```

될 것 같지만 메시지가 **Employee is not abstract and does not override abstract method getId() in Identified** 라고 바뀔 뿐 여전히 컴파일은 되지 않는다. `Person`과 `Identified` 인터페이스가 동일한 메서드를 갖고 있긴 하지만 컴파일러 입장에서는 두 개가 정말 같은 목적의 메서드인지 알 길이 없다. 따라서 같은 모양의 메서드지만 두 메서드를 다른 것으로 보고 클래스가 `Identified`를 구현하지 않았다고 판단한다.(개인적으로는 이것이 일관성이 부족하다고 생각하는데 그 이유는 두 인터페이스 모두 기본 메서드를 구현하지 않는 경우에는 충돌이 일어나지 않기 때문이다.)

가장 좋은 해결 방안은 충돌이 나는 경우를 만들지 않는 것이다. 두 인터페이스가 동일한 메서드를 갖고 있다면 인터페이스 간에 상속 관계가 있지는 않은지, 메서드 이름을 너무 포괄적으로 정한 것은 아닌지 따져보고 충돌 상황을 피하는 게 좋다.

메서드 이름이나 기본 메서드 구현을 포기하지 않고 컴파일 에러를 해결하는 방법은 아래와 같다.

## 클래스에서 충돌 메서드 구현

가장 단순한 방법으로 클래스에서 충돌 메서드의 구현을 덮어 버리는 것이다. 이 때 클래스에서 구현을 새로 할 수도 있지만 어느 한 쪽의 기본 메서드를 사용할 수도 있다.

```java
public class Employee implements Person, Identified {
    @Override
    public int getId() {
        // Person의 기본 메서드를 사용하고 싶은 경우.
        // Identified의 기본 메서드를 사용하려면 Identified.super.getId()를 반환한다.
        return Person.super.getId();
    }
}
```

## 인터페이스간 상속

`Person` 인터페이스가 `Identified`를 상속받도록 하면 `Employee`에 구현 메서드가 없어도 문제를 해결할 수 있다. 하지만 이런 결정을 하기 전에 인터페이스 사이의 관계를 잘 고려해야한다.

```java
public interface Person extends Identified {
    String getName();
    default int getId() { return 0; }
}
```
```java
public interface Identified {
    default int getId() { return Math.abs(hashCode()); }
}
```
이렇게 하면 `Employee`는 결국 `Person`의 기본 메서드를 사용하게 된다.
