---
title: Java의 고유 락(intrinsic lock)에 대해
date: 2017-10-04 22:00
categories:
- Java
tags:
- java
- lock
- concurrency
---

## 고유 락과 `synchronized` 블록
자바의 모든 객체는 락(lock)을 갖고 있다. 모든 객체가 갖고 있으니 고유 락(intrinsic lock)이라고도 하고, [모니터][1]처럼 동작한다고 하여 모니터 락(monitor lock) 혹은 그냥 모니터(monitor)라고도 한다. 자바의 `synchronized` 블록은 동시성 문제를 해결하는 가장 간편한 방법으로, 고유 락을 이용하여 여러 스레드의 접근을 제어한다. 간단한 예제를 통해 `synchronized`의 사용법을 알아보자.<!-- more -->

```java
public class Counter {
  private int count;

  public int increase() {
    return ++count; // 스레드 안전하지 않은 연산.
  }
}
```

`Counter` 클래스는 숫자를 셀 때 사용하는 클래스로 `increase()`를 호출할 때마다 `count` 변수를 1만큼 증가시키고, 그 값을 반환한다. `++count` 문은 한 줄 짜리 코드로 원자적(atomic)으로 동작할 것 같지만 사실은 그렇지 않다. `++` 연산자를 호출했을 때 실제로는 세 가지 연산이 순차적으로 수행된다. 변수를 메모리에서 읽고, 증가시키고, 다시 메모리에 쓴다. 이는 동시성 프로그래밍에서 문제가 되는 전형적인 read-modify-write 패턴이다. 두 스레드가 동시에 같은 값을 읽고, 값을 증가시켜 저장하면 `increase()` 호출 횟수와 `count` 값에 차이가 발생한다. 동시성 문제는 결국 여러 스레드가 공유 자원으로 접근하기 때문에 발생한다. 여기서 공유 자원은 `count` 변수이다. 동시성 문제를 해결하기 위해 `count` 변수로 접근하는 스레드를 제어해야 한다. 이제 고유 락 즉, `synchronized` 블록을 이용하여 `Counter` 클래스를 스레드 안전하게 만들어 보자.

```java
public class Counter {
  private Object lock = new Object();
  private int count;

  public int increase() {
    synchronized(lock) {
      return ++count;
    }
  }
}
```

`lock`이라는 `Object` 인스턴스를 이용하여 여러 스레드가 동시에 `count` 변수에 접근하지 못하도록 제어했다. 이제 `increase()` 메서드는 한 번에 한 스레드만 실행할 수 있다. 한 스레드가 먼저 락을 획득한 경우 다른 스레드는 기다려야 한다. 그리고, 사실 이 코드에서 락을 위해 별도의 객체를 생성할 필요가 없다. `Counter`의 인스턴스도 자바 객체이므로 락으로 사용할 수 있다.

```java
public class Counter {
  private int count;

  public int increase() {
    synchronized(this) {
      return ++count;
    }
  }
}
```

`this`를 이용하여 별도의 락 생성 없이 `synchronized` 블록을 구현하였다. 하지만 좀 더 쉽게 구현할 수 있다. 예제의 `synchronized` 블록은 `increase()` 메서드 전체를 감싸고 있는데, 이런 경우에는 메서드에 `synchronized` 키워드를 붙여주는 것으로 대신할 수 있다.

```java
public class Counter {
  private int count;

  public synchronized int increase() {
    return ++count;
  }
}
```

## 재진입 가능성(Reentrancy)
자바의 고유 락은 재진입 가능하다. 재진입 가능하다는 것은 락의 획득이 호출 단위가 아닌 스레드 단위로 일어난다는 것을 의미한다. 이미 락을 획득한 스레드는 같은 락을 얻기 위해 대기할 필요 없다. 이미 락을 갖고 있으므로 같은 락에 대한 `synchronized` 블록을 만났을 때 대기없이 통과한다.

```java
public class Reentrancy {
  public synchronized void a() {
    System.out.println("a");
    // b가 synchronized로 선언되어 있지만 a진입시 이미 락을 획득하였으므로,
    // b를 호출할 수 있다.
    b();
  }

  public synchronized void b() {
    System.out.println("b");
  }

  public static void main(String[] args) {
    new Reentrancy().a();
  }
}
```

만약 자바의 고유 락이 재진입 가능하지 않다면 위 코드는 a 메서드 내부의 b를 호출하는 지점에서 데드락이 발생한다.

## 구조적인 락(structured lock)
고유 락을 이용한 동기화를 구조적인 락(structured lock)이라고 한다. `synchronized` 블록 단위로 락의 획득/해제가 일어나므로 구조적이라고 한다. `synchronized` 블록을 진입할 때 락의 획득이 일어나고, 블록을 벗어날 때 락의 해제가 일어난다. 따라서 구조적인 락 A와 B가 있을 때 A 획득 -> B 획득 -> B 해제 -> A 해제는 가능하지만 A 획득 -> B 획득 -> A 해제 -> B 해제는 불가능 하다. 이런 순서로 락을 사용해야 하는 경우라면 [ReentrantLock][2]과 같은 명시적인 락을 사용해야 한다.

## 가시성(visibility)
동시성 프로그램의 이슈 중 하나는 가시성이다. `synchronized`를 적용하기 이전의 `Counter` 예제로 돌아가보자. 이 코드는 두 스레드가 절대로 동시에 `increase()`를 호출하는 일이 없다고 하더라도 문제가 있다. 한 스레드가 쓴 값을 다른 스레드가 볼 수도 있고 그렇지 않을 수도 있기 때문이다. 이를 가시성 문제라고 한다. 이 문제의 원인은 다양하다. 최적화를 위해 컴파일러나 CPU에서 발생하는 코드 재배열(Reordering)때문에 이런 문제가 발생할 수도 있고, 멀티 코어 환경에서는 코어의 캐시 값이 메모리에 제때 쓰이지 않아 문제가 발생할 수도 있다.
락을 사용하면 가시성의 문제가 사라질까? 그렇다. 자바에서는 스레드가 락을 획득하는 경우 그 이전에 쓰였던 값들의 가시성을 보장한다. `synchronized`가 적용된 `Counter` 예제에서 스레드 A, 스레드 B 순서로 `increase()`를 호출했을 때, 스레드 B는 스레드 A가 쓴 값을 읽을 수 있다(visible 하다). 이는 고유 락 뿐만 아니라 [ReentrantLock][2] 같은 명시적인 락에서도 똑같이 적용된다.

[1]: https://en.wikipedia.org/wiki/Monitor_(synchronization)
[2]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReentrantLock.html
