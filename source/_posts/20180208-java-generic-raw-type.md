---
title: Java 제네릭 - Raw Type을 쓰면 안되는 이유
date: 2018-02-08 17:00
categories:
- Java
tags:
- java
- generic
---

작년 여름에 자바 제네릭과 가변성에 대해 정리하고, 나름 깨달은 바가 있다 생각했지만 최근에 아직도 모르는 게 많다는 것을 알게 되었다. 나의 무지를 일깨워 준 것은 프로젝트 진행 중에 맞닥뜨린 컴파일 타임 에러(당시에는 도무지 이해가 되지 않았던;)였으니...<!-- more -->

## 당황스러웠던 컴파일 타임 에러
같이 프로젝트를 진행하던 분이 나를 부르더니 컴파일이 안된다며 코드를 보여주었다. 아래 코드 예제는 당시에 마주쳤던 컴파일 에러를 그대로 재현한다.

```java
public class Trouble<T> {
    public List<String> getStrs() {
        return Arrays.asList("str");
    }

    public static void main(String[] args) {
        Trouble g = new Trouble();

        for (String str : g.getStrs()) {
            System.out.println(str);
        }
    }
}
```
이 코드를 컴파일 하려고 하면 다음과 같은 컴파일 에러가 발생한다.

```
Generic.java:17: error: incompatible types: Object cannot be converted to String
        for (String str : g.getStrs()) {
                                   ^
```
컴파일 에러가 나는 코드는 for 문에서 요소를 `String str` 변수로 받는 부분이다. 응? `getStrs()`는 `List<String>`을 반환하는데?!

## Raw Type
[Raw Type][2]은 타입 파라미터가 없는 제네릭 타입을 의미한다. 예제 코드의 `g`가 바로 Raw Type 변수이다. `Trouble` 클래스는 제네릭 타입으로 정의되었지만 변수 `g`는 타입 파라미터 없이 정의되었다. 애초에 제네릭으로 정의되지 않은 클래스나 인터페이스에는 Raw Type이란게 없다.

Raw Type을 사용하면 왜 위와 같은 컴파일 에러가 발생하는 것일까? 해답은 [JLS 4.8][3]에 나와있다. 이 문서를 보면 온통 어려운 얘기들 뿐인데 그 중에 이런 문장이 있다.

> The type of a constructor, instance method, or non-static field of a raw type C that is not inherited from its superclasses or superinterfaces is the raw type that corresponds to the erasure of its type in the generic declaration corresponding to C.

번역하면 이렇다. **"Raw Type의 생성자, 메서드, 인스턴스 필드는 모두 Raw Type이다."** (그런거였어..? ㄷㄷ) Raw Type은 타입 파라미터 `T`만 지워버리는 것이 아니라 해당 인스턴스와 연관된 모든 타입 파라미터를 지워버린다. 그래서 `g.getStrs()`의 반환 타입이 `List<String>`이 아닌 Raw Type `List`가 된 것이다.

자바와 같은 정적 타입 언어의 강점은 프로그램을 실행하기 전에 컴파일 에러를 잡을 수 있다는 것이다. 하지만 Raw Type은 부주의하게 사용하면 런타임 에러를 일으킬 수 있다. 아래 코드는 런타임 에러를 발생시키는 예제이다.

```java
List<String> good = new ArrayList<>();
List bad = good;
// warning: unchecked call to add(E) as a member of the raw type List
bad.add(1);
for (String str : good) {
    System.out.println(str);
}
```

경고가 발생하긴 하지만 컴파일이 되는 코드이다. 하지만 이 코드를 실행하면 `java.lang.ClassCastException`이 발생한다.

애초에 Raw Type은 자바에 제네릭이 도입되기 전(JDK 5.0 이전) 코드와 호환성을 보장하기 위한 것이다. 정적 타입 언어라는 자바의 강점을 이용하기 위해서 Raw Type을 사용하지 말아야 한다.

## 참고 문서
[Java Tutorial - Raw Type][2]
[Java Language Specification 4.8][3]
[Stackoverflow - What is a raw type and why shouldn't we use it?][4]

[1]: /2017/05/21/java-generic-and-variance-1/
[2]: https://docs.oracle.com/javase/tutorial/java/generics/rawTypes.html
[3]: https://docs.oracle.com/javase/specs/jls/se8/html/jls-4.html#jls-4.8
[4]: https://stackoverflow.com/questions/2770321/what-is-a-raw-type-and-why-shouldnt-we-use-it
