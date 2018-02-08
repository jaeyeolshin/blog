---
title: Java 제네릭과 가변성(variance) 1편
date: 2017-05-21 01:30
categories:
- Java
tags:
- java
- generic
---

[카이 호스트만의 코어 자바8][1] 책으로 공부를 계속하고 있다. 게으른 탓인지 진도가 정말 느리다. 간만에 배운 내용을 정리하고자 했는데 직전에 쓴 글이 한 달 전이라니... 너무 부끄럽다. 가끔 yes24에서 서평들을 보면 일주일 만에 기술 책 한권을 독파하고 소감을 쓰시는 분들도 계신데, 이런 분들을 보면 부럽기도 하고 존경스럽다. 여하튼 이번 주제는 Java의 제네릭과 가변성(variance)이다. 제네릭을 자주 사용하면서도 자바 제네릭의 타입 경계, 와일드 카드, 타입 소거 같은 특징들에 대해서는 잘 알지 못했다. 가변성은 볼 때마다 헷갈리는 개념이다. 이번 포스팅을 통해 이런 개념들을 정리해보려고 한다.<!-- more -->

# 제네릭 클래스
제네릭 클래스는 타입 파라미터를 한 개 이상 받는 클래스이다. 타입을 파라미터로 받는다는 것은 클래스 선언시에는 타입을 특정하지 않고 인스턴스 생성시에 사용할 타입을 정하는 것을 의미한다. 보통 제네릭은 컬렉션 클래스와 많이 사용된다. 컬렉션 중 타입 파라미터를 하나만 받는 대표적인 인터페이스로는 `java.util.List`가 있고, 두 개 받는 것으로는 `java.util.Map`이 있다. 리스트는 저장되는 요소의 타입을 위해 타입 파라미터를 하나만 받는다. 맵은 키-값의 컬렉션으로 키의 타입, 값의 타입 두 개의 타입 파라미터를 받는다. 타입 파라미터는 인스턴스 변수, 메서드의 파라미터 및 반환 타입으로 사용된다. 자바 개발자들은 제네릭을 사용해서 보다 편리하고 안전하게 프로그래밍을 할 수 있다. 아래는 흔히 볼 수 있는 제네릭을 이용한 코드이다.

```java
List<String> cities = new ArrayList<>();
cities.add("Changwon");
cities.add("Seoul");
cities.add("Suwon");
cities.add("Yongin");

for (String city : cities) {
    System.out.println(city.toUpperCase());
}
// 결과:
// CHANGWON
// SEOUL
// SUWON
// YONGIN
```

제네릭은 자바의 타입을 확장하기 때문에 잘못된 입력을 컴파일 타임에 잡아준다. 즉, 프로그래머가 `List<String>`라고 자료형을 선언했다면 이 리스트에는 문자열만 저장하고, 조회할 수 있다는 확신을 가질 수 있다. 위 코드 예제에서 `cities.add(1);`과 같은 코드는 컴파일 에러를 발생시킨다. 그 이유는 `List` 인터페이스에 `add()` 메서드가 아래와 같이 제네릭 타입으로 정의되어 있고, `List<String>`은 타입 파라미터 `E`를 `String`으로 치환하기 때문이다. 따라서 `List<String>`의 `add()`는 문자열을 입력받는 메서드가 된다.

```java
public interface List<E> extends Collection<E> {
    ...
    boolean add(E e); // List<String>에서는 boolean add(String e); 가 된다.
    ...
}
```

`cities`를 향상된 for loop으로 각 문자열 요소를 순회할 수 있는 것도 같은 원리이다. `List<E>`는 `Collection<E>`를 `Collection<E>`는 `Iterable<E>`를 상속받는다. 이 상속 관계는 타입 파라미터와 상관없이 유지된다. 결국 `List<String>`은 `Iterable<String>`이기 때문에 개별 문자열을 향상된 for loop으로 순회할 수 있는 것이다. 자바의 제네릭은 2002년 Java 5 발표와 함께 등장했는데 그 이전에는 컬렉션을 다루는 것이 무척이나 불편했다고 한다.(물론 나는 그 시절의 자바를 겪어보지 않았다.) 제네릭이 없었기 때문에 요소의 타입은 최상위 클래스인 `Object`였다. Iterator는 Java 5 이전에도 있었지만 향상된 for loop은 Java 5에서 소개되었다. 아마도 그 시절의 리스트 순회는 다음과 같지 않았을까?

```java
List cities = new ArrayList();
// (요소를 삽입한다.)
for (Iterator iter = cities.iterator(); iter.hasNext(); ) {
    String city = (String) iter.next();
    System.out.println(city.toUpperCase());
}
```

# 제네릭 메서드
제네릭 메서드는 타입 파라미터를 받는 메서드이다. 제네릭 메서드는 일반 클래스나 제네릭 클래스의 메서드가 될 수 있다. 다음 제네릭 메서드는 리스트의 처음 세 요소만 갖는 새로운 리스트를 반환한다.

```java
public static <T> List<T> firstThree(List<T> list) {
    return list.stream().limit(3).collect(Collectors.toList());
}
```

이 메서드를 앞선 예제 코드에 응용해보자.

```java
List<String> cities = new ArrayList<>();
cities.add("Changwon");
cities.add("Seoul");
cities.add("Suwon");
cities.add("Yongin");

for (String city : firstThree(cities)) {
    System.out.println(city.toUpperCase());
}
// 결과:
// CHANGWON
// SEOUL
// SUWON
```

향상된 for loop 안에서 제네릭 메서드 `firstThree()`를 호출하였다. 제네릭 메서드를 호출할 때는 타입 파라미터를 명시하지 않아도 컴파일러가 자동으로 타입을 유추한다. 타입 파라미터 `T`는 자동으로 `String`으로 치환되고, 메서드는 `List<String>`을 반환한다.

# 타입 경계
제네릭 클래스나 메서드가 받는 타입 파라미터를 제한하고 싶은 경우에 타입 경계를 사용할 수 있다. `firstThree()`는 전달 받은 타입 파라미터에 구애 받지 않는 메서드이기 때문에 타입을 제한하지 않았다. 하지만 특정 타입의 메서드나 필드를 이용해야 하는 경우라면 그에 맞는 경계를 선언해야 한다. 다음 메서드는 문자열의 리스트를 입력받아 각 문자열의 첫 번째 글자로 이루어진 리스트를 반환한다.

```java
public static <T extends CharSequence> List<Character> firstChars(List<T> list) {
    return list.stream().map(cs -> cs.charAt(0)).collect(Collectors.toList());
}
```

타입 파라미터 `<T extedns CharSequence>`는 타입 파라미터 `T`를 `CharSequence`의 서브타입으로 제한한다. 따라서 `firstChars()`를 호출할 때는 `CharSequence` 혹은 그 서브타입의 리스트를 인자로 전달해야 한다. 이제 `T`가 `CharSequence`인 것을 알았으므로 리스트 각 요소에 대해 `CharSequence`의 메서드를 호출할 수 있다. 예제에서는 `charAt(0)`을 호출하여 첫 번째 문자를 반환하도록 하였다. 타입 파라미터에 경계를 설정하지 않는 것은 최상위 클래스인 `Object`를 경계로 설정하는 것과 동일하다.

# 타입 가변성(variance)과 와일드 카드
![자바의 와일드카드 서브타이핑](/images/java-generic-and-variance-1/Java_wildcard_subtyping.svg)
<p style="text-align:center;color:#808080;">자바의 와일드카드와 클래스 계층(이미지 출처: [위키백과][2])</p>

## 가변성이란?
가변성은 타입 파라미터가 클래스 계층에 어떤 영향을 미치는지를 나타낸다. 다음 가변성 표를 보자.

| | **의미** |
| --- | --- |
| **공변성(covariant)** | T'가 T의 서브타입이면, C&lt;T'&gt;는 C&lt;T&gt;의 서브타입이다. |
| **반공변성(contravariant)** | T'가 T의 서브타입이면, C&lt;T&gt;는 C&lt;T'&gt;의 서브타입이다. |
| **무변성(invariant)** | C&lt;T&gt;와 C&lt;T'&gt;는 아무 관계가 없다. |

자바의 제네릭은 기본적으로 무변성이다. `String`은 `CharSequence`의 서브타입이지만, `List<String>`과 `List<CharSequence>`는 아무 관계가 없다. `List<CharSequence> list = new ArrayList<String>();` 같은 코드는 컴파일 에러를 발생한다. 그렇기 때문에 위 표에서 공변성, 반공변성을 설명한 것은 틀린 표현이다. 표는 [타입과 다형성의 기초][3]에서 설명한 스칼라의 가변성 표를 그대로 가져와 꺽쇠 모양만 자바에 맞게 바꾼 것이다. 그럼에도 위와 같이 표기한 이유는 개념적으로 이해 하기가 더 쉽기 때문이다. 실제로 자바 제네릭에서는 와일드카드를 사용해야만 가변성을 지정할 수 있다. 와일드카드를 설명한 후에 실제 자바 표현에 맞게 위 표를 수정해 볼 것이다.

## 와일드카드
와일드 카드를 이해하기 위해 다음 두 메서드를 보자. 어느 것이 더 유용한 메서드일까?

```java
public static void printCollection(Collection collection) {
    for (Object e : collection) {
        System.out.println(e);
    }
}

public static void printCollectionGen(Collection<Object> collection) {
    for (Object e : collection) {
        System.out.println(e);
    }
}
```

`printCollectionGen()`이 제네릭도 사용하고 뭔가 있어 보이지만 이 메서드는 별로 쓸모가 없다. 앞서 말했듯이 자바 제네릭은 기본적으로 무변성이기 때문이다. `Object`가 최상위 클래스이긴 하지만 `Collection<Object>`는 `Collection<String>`, `List<String>`과는 아무 관계가 없다.

```java
List<String> cities = new ArrayList<>();
cities.add("Changwon");
cities.add("Seoul");
cities.add("Suwon");
cities.add("Yongin");

printCollection(cities); // List<String>은 Collection의 서브타입이다.
// 결과:
// Changwon
// Seoul
// Suwon
// Yongin
printCollectionGen(cities); // List<String>은 Collection<Object>와 아무 관계가 없다.
// 컴파일 에러
```

`printCollectionGen()`을 쓸모있게 만드려면 타입 파라미터에 `Object` 대신 와일드카드 `?`를 사용해야 한다.

```java
public static void printCollectionGen(Collection<?> collection) {
    for (Object e : collection) {
        System.out.println(e);
    }
}
```

이렇게 하면 위에서 컴파일 에러가 나던 코드도 정상적으로 동작한다. 와일드카드 `?`는 *Unknown* 타입으로 어떤 타입이든 올 수 있다. `?`가 무엇인지는 모르겠지만 `Object`인 것은 확실하기 때문에 `Object` 타입으로 요소를 순회할 수 있다.

## 서브타입 와일드카드
타입 파라미터에서 그랬던 것 처럼 와일드카드에도 타입 경계를 설정할 수 있다. 와일드카드를 사용하면 `firstChars()` 메서드를 좀 더 간단하게 작성할 수 있다.

```java
public static List<Character> firstChars(List<? extends CharSequence> list) {
    return list.stream().map(cs -> cs.charAt(0)).collect(Collectors.toList());
}
```

자바는 서브타입 와일드카드를 이용하여 <strong>공변성</strong>을 표현한다. 이 메서드는 `CharSequence`의 서브타입으로 이루어진 리스트 객체라면 무엇이든지 받을 수 있다(예를 들면 `ArrayList<String>`). 서브타입 와일드카드는 메서드의 반환 타입에서는 사용할 수 있지만, 메서드의 파라미터에는 사용할 수 없다. 다음 코드를 보자.

```java
List<? extends String> list = new ArrayList<String>();
String first = list.get(0); // 반환 타입을 String 변수에 저장할 수 있다.
list.add("abc"); // 컴파일 에러. String 변수를 인자로 넘길 수 없다.
```

`list`의 타입 파라미터는 `? extends String` 이다. IntelliJ에서 `list.get()`이 반환하는 타입을 보면 `capture of ? extends String`라고 나타난다. 뭔지는 모르겠지만 `String`의 서브타입이라는 것이다. 그렇기 때문에 `get()`의 반환 값을 `String` 변수에 저장할 수 있다. 하지만 `add()`는 사용할 수 없다. `add()`를 호출하려면 `capture of ? extends String`나 그 서브타입 변수를 넘겨야 하지만 뭔지도 모르는 타입의 변수를 만들 방법이 없다. `"abc"`는 `capture of ? extends String`이 아니기 때문에 컴파일 에러가 발생하는 것이다.

## 슈퍼타입 와일드카드
서브타입 와일드카드가 공변성을 나타낸다면 슈퍼타입 와일드카드는 <strong>반공변성</strong>을 나타낸다. 반공변성의 의미를 다시 살펴 보자.

> T'가 T의 서브타입이면, C&lt;T&gt;는 C&lt;T'&gt;의 서브타입이다.

타입 파라미터와 계층 관계가 반대로 움직인다니 뭔가 이상하다. 그리고 이런게 어디에 쓰인단 말인가. 앞서 서브타입 와일드카드는 메서드의 반환 값에는 사용될 수 있지만 파라미터 타입에서는 사용될 수 없다고 했다. 슈퍼타입 와일드카드는 메서드의 파라미터 타입을 설정할 때 사용된다. 하지만 반대로 메서드의 반환 값으로는 사용할 수 없다. 앞선 예제의 `? extends String`을 `? super String`으로 바꾸고 무슨 일이 일어나는지 살펴보자.

```java
List<? super String> list = new ArrayList<String>();
String first = list.get(0); // 컴파일 에러. 반환 타입을 String 변수에 저장할 수 없다.
list.add("abc"); // String 변수를 인자로 넘길 수 있다.
```

신기하게도 컴파일 에러가 나는 코드가 뒤바뀌었다. 서브타입 와일드카드에서 했던 것처럼 왜그런지 한번 살펴보자. 이제 `list`의 타입 파라미터는 `? super String`이 되었다. `get()`이 반환하는 타입은 뭔지 모르겠지만 `String`의 슈퍼타입이기 때문에 `String` 타입의 변수에 저장할 수 없다. 이는 `String` 변수에 `new Object()`를 할당할 수 없는 것과 같은 이치다. 반대로 `? super String`가 뭔지는 모르지만 `String`의 슈퍼타입인 것은 확실하므로 `String` 변수를 `add()` 호출에 사용할 수 있는 것이다. 이를 통해 **메서드의 반환 타입은 공변적이고, 메서드의 파라미터 타입은 반공변적** 이라는 것을 알 수 있다.

이런 이유로 슈퍼타입 와일드카드는 주로 인자 타입이 타입 파라미터로 결정되는 함수형 인터페이스에서 사용된다. 전형적인 예로 함수형 인터페이스 `Predicate<T>`가 있다. `Predicate<T>`는 `T` 타입의 변수를 입력받아 불리언 값을 반환한다. 다음 예제를 보자.

```java
Stream<String> stream = Stream.of("가", null, "나", null, "다");
Predicate<Object> notnull = (obj) -> obj != null;
stream.filter(notnull).forEach(System.out::print);
// 결과:
// 가나다
```

`Stream<String>.filter()`는 `Predicate<? super String>`을 인자로 받는다. 인자로 받은 함수를 이용하여 문자열을 테스트하고 테스트를 통과한 문자열로만 구성된 스트림을 반환한다. `Predicate<Object>`는 `Predicate<? super String>`의 서브타입이기 때문에 `notnull`을 인자로 넘겨줄 수 있는 것이다.

자, 이제 위에서 약속한대로 가변성 표를 자바 표기법에 맞게 바꾸어보자.

| | **의미** |
| --- | --- |
| **공변성(covariant)** | T'가 T의 서브타입이면, C&lt;T'&gt;는 C&lt;? extends T&gt;의 서브타입이다. |
| **반공변성(contravariant)** | T'가 T의 서브타입이면, C&lt;T&gt;는 C&lt;? super T'&gt;의 서브타입이다. |
| **무공변성(invariant)** | C&lt;T&gt;와 C&lt;T'&gt;는 아무 관계가 없다. |

# 참고 문서
[카이 호스트만의 코어 자바8][1]
[Covariance and contravariance - 위키피디아][2]
[타입과 다형성의 기초 - twitter.github.io][3]
[wildcards - 오라클](https://docs.oracle.com/javase/tutorial/extra/generics/wildcards.html)

[1]: http://www.yes24.com/24/goods/23449538?scode=032&OzSrank=1
[2]: https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)#Use-site_variance_annotations_.28wildcards.29
[3]: https://twitter.github.io/scala_school/ko/type-basics.html
