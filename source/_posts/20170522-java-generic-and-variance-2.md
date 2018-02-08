---
title: Java 제네릭과 가변성(variance) 2편
date: 2017-05-22 12:20
categories:
- Java
tags:
- java
- generic
---

[1편][1]을 쓰느라 진을 너무 많이 뺐다. 사실 이 블로그를 쓰는 이유는 공부하고 까먹고 다시 공부하는 게 싫어서인데 쓰다보면 욕심이 생겨서 다른 자료들도 찾아보고, 예제 코드도 스스로 만들어 보게 된다. 이런 노력이 미래의 내가(그리고 우연히 이곳을 들른 여러분들이?!) 정리한 내용을 이해하는 데 도움이 되었으면 좋겠다. 이번 글에서는 [카이 호스트만의 코어 자바8][2]의 내용과 상관없이 [1편][1]에서 다뤘던 가변성에 대한 내용을 조금만 더 정리하려 한다.<!-- more -->

# 리스코프 치환 원칙(Liskov Substitution Principle)
객체 지향 프로그래밍을 공부해본 사람들이라면 한번쯤은 들어봤을 법한 원칙이다. 줄여서 LSP라고도 하는 이 원칙은 수학적인 내용을 담고 있을 것 같아 압도되는 느낌을 주지만, 사실 OOP에서는 아주 친숙한 개념이다. [위키피디아][3]에는 이 원칙을 이렇게 설명하고 있다.

> 컴퓨터 프로그램에서 자료형 **<em>S</em>** 가 자료형 **<em>T</em>** 의 하위형이라면 필요한 프로그램의 속성(정확성, 수행하는 업무 등)의 변경 없이 자료형 **<em>T</em>** 의 객체를 자료형 **<em>S</em>** 의 객체로 교체(치환)할 수 있어야 한다.

흔히 알고 있는 상속, 다형성의 개념과 일치한다. 하지만 상속을 사용한다고 해서 무조건 LSP를 지키는 것은 아니다. [위키피디아][3]를 보면 LSP를 위반하여 상속을 하는 사례가 나오는데 한 번 읽어보면 좋다.

자바 가변성 얘기를 하다말고 갑자기 LSP 얘기를 꺼내서 조금 의아하게 느껴질 수도 있겠다. 하지만 가변성을 공부할 때 이 원칙에 입각하여 생각해보면 이해하기가 쉬워진다. 나는 가변성의 개념을 스칼라를 공부할 때 처음 접하게 되었는데, 당시에 반공변성을 이해하는 것이 너무나도 힘들었다. 오늘 겨우겨우 이해를 해도 다음 날이 되면 또 납득이 되질 않아서 고민하는 식이었다. 여기서 다시 반공변성의 의미를 떠올려보자.

> T'가 T의 서브타입이면, C&lt;T&gt;는 C&lt;T'&gt;의 서브타입이다.

타입 파라미터와 반대 방향으로 클래스 계층이 결정된다는 것이 이해하기 어렵다. 트위터의 스칼라 문서인 [타입과 다형성의 기초][4]에서는 `Animal`, `Bird`, `Chicken`, `Duck` 같은 임의의 클래스 계층을 도입하여 '오리는 닭짓(?)을 할 수 없기 때문에 메서드 파라미터는 반공변적이다' 라고 친절하게 설명하지만 머리 속에 클래스들이 둥둥 떠다닐 뿐 잘 와닿지 않았다. 하지만 리스코프 치환 원칙에서 서브타입의 의미를 생각해보면 쉽게 이해할 수 있다. 자바의 `Predicate<T>`를 예로 들어보자. [1편][1]에서 설명했다시피 `Predicate<T>`는 타입 `T`가 메서드의 파라미터가 되는 함수형 인터페이스이므로 반공변적이다. `CharSequence`에 대해 테스트할 수 있는 함수를 저장하고 싶다면 다음과 같이 변수를 선언해야 한다.

```java
Predicate<? super CharSequence> pred;
```

그리고 `pred` 변수에 저장할 수 있는 다음 세 가지 후보를 생각해보자.

```java
Predicate<CharSequence> predCs = cs -> cs.charAt(0) == 'a';
Predicate<String> predStr = str -> "ABC".equals(str.toUpperCase());
Predicate<Object> predObj = obj -> obj != null;
```

우리가 기대하는 `pred` 변수의 속성은 무엇인가? <strong>`CharSequence` 타입의 인자를 넘겨 호출하는 것</strong>이다. `Predicate<? super CharSequence>`의 서브타입이라면 이 속성을 유지해야 한다. 그러면 누가 `pred` 변수에 들어갈 수 있고 없는지가 명확히 구분된다.

| 후보 | `CharSequence` 타입의 인자로 호출할 수 있는가? |
| --- |:---:|
| `Predicate<CharSequence> predCs` | O |
| `Predicate<String> predStr` | X |
| `Predicate<Object> predObj` | O |

# 사용처 가변성(use-site variance)과 선언처 가변성(declaration-site variance)
자바는 제네릭 클래스를 사용하는 쪽에서 와일드카드로 가변성을 결정하기 때문에 자바의 가변성을 <strong>사용처 가변성</strong>이라고 한다. 이는 개발자를 조금 귀찮게 하는데 매번 변수나 파라미터의 타입을 선언할 때마다 공변적 혹은 반공변적이어야 하는지 고민해야 하고, 타입 파라미터에는 타입 뿐만 아니라 `<? extends T>`, `<? super T>`와 같이 와일드카드와 경계를 함께 적어줘야 하기 때문이다. 하지만 어떤 제네릭 클래스든 개발자가 자기가 사용하고 싶은대로 가변성을 결정할 수 있다는 장점도 갖고 있다. 예를 들어 `List<T>`는 본래 요소의 조회 및 추가가 모두 가능한 클래스이지만 타입을 `List<? extends String>`로 지정하면 읽기 전용 리스트로 사용할 수 있다. 아무짝에도 쓸모가 없겠지만 `Predicate<? extends CharSequence>`와 같은 타입도 선언이 가능하다.

사용처 가변성과 대응되는 개념이 바로 <strong>선언처 가변성</strong>이다. 여기서 선언처는 타입이 선언되는 곳을 의미한다. 스칼라는 대표적인 선언처 가변성 언어이다. 파라미터를 하나만 받는 함수를 나타내는 `Function1` 트레이트(자바의 인터페이스와 유사)를 살펴보자.

```scala
trait Function1[-T1, +R] extends AnyRef
```

`AnyRef`는 자바의 `Object`와 동일하다. `T1`, `R`은 각각 파라미터와 리턴 값에 해당하는 타입 파라미터이다. 그런데 앞에 `-`, `+`가 붙어있다. 스칼라에서는 타입 파라미터 앞에 `-`를 붙여 반공변성, `+`를 붙여 공변성을 나타낸다. 이처럼 스칼라에서는 타입 선언부에 가변성이 정해져있기 때문에 사용하는 입장에서는 편리하다. 앞서 봤던 `Predicate<T>` 예제를 스칼라 버전으로 바꾸어 보자.

```scala
var pred: Function1[CharSequence, Boolean]
val predCs: Function1[CharSequence, Boolean] = cs => cs.charAt(0) == 'a'
val predStr: Function1[String, Boolean] = str => "ABC" == str.toUpperCase
val predObj: Function1[AnyRef, Boolean] = ref => ref != null

pred = predCs
pred = predStr // 컴파일 에러
pred = predObj
```

스칼라는 `Function1` 정의에서 가변성이 함께 정의되어 있기 때문에 개발자가 가변성에 대한 고민을 하지 않아도 되고, 번거롭게 `? extends`, `? super`를 쓰지 않아도 된다. 대신 스칼라는 라이브러리 개발자들이 조금 더 고생한다. 스칼라 컬렉션 라이브러리는 `scala.collection.mutable` 패키지와 `scala.collection.immutable` 패키지로 나뉜다. `mutable` 컬렉션은 조회와 수정 모두 가능하기 때문에 무공변적이지만 `immutable` 컬렉션은 읽기 전용이므로 공변적이다.

# 참고 문서
[Covariance and contravariance - 위키피디아][1]
[카이 호스트만의 코어 자바8][2]
[리스코프의 치환 원칙][3]
[타입과 다형성의 기초 - twitter.github.io][4]

[1]: /2017-05-21/java-generic-and-variance-1/
[2]: http://www.yes24.com/24/goods/23449538?scode=032&OzSrank=1
[3]: https://ko.wikipedia.org/wiki/%EB%A6%AC%EC%8A%A4%EC%BD%94%ED%94%84_%EC%B9%98%ED%99%98_%EC%9B%90%EC%B9%99
[4]: https://twitter.github.io/scala_school/ko/type-basics.html
