---
title: Java Object 클래스의 wait과 notify의 사용법
date: 2017-10-05 21:50
categories:
- Java
tags:
- java
- concurrency
---

## [Object][1]의 wait, notify와 notifyAll
자바의 최상위 클래스인 `Object`에는 몇 가지 메서드가 존재한다. 널리 쓰이는 `toString()`은 객체를 문자열로 표현할 때, `hashCode()`는 객체의 해시 값을 계산할 때 사용된다. 거의 사용되지 않고 가끔 IDE의 자동 완성 기능에서나 보게 되는 메서드들도 있으니 그것이 바로 `wait()`, `notify()`, `notifyAll()`이다. 이들의 동작을 간략히 정리하면 다음과 같다.<!-- more -->

| 메서드 | 기능 | 비고 |
| --- | --- | --- |
| wait | 갖고 있던 고유 락을 해제하고, 스레드를 잠들게 한다. | 호출하는 스레드가 반드시 고유 락을 갖고 있어야 한다. 다시 말해, `synchronized` 블록 내에서 호출되어야 한다. |
| notify | 잠들어 있던 스레드 중 임의로 하나를 골라 깨운다. | 상동 |
| notifyAll | 호출로 잠들어 있던 스레드 모두 깨운다. | 상동 |

![자동 완성에나 나오던 메서드인 줄 알았는데...](/images/java-object-wait-and-notify/waitnotify_autocomplete.png)

세 메서드가 공통으로 갖는 전제 조건이 보인다. 그것은 호출 스레드가 반드시 대상 객체의 고유 락을 갖고 있어야 한다는 것이다. 다시 말해, 이 메서드들은 `synchronized` 블록 내에서 실행되어야 한다.(`synchronized`에 대한 요약은 [여기서][2] 볼 수 있다!) 고유 락을 획득하지 않은 상태에서 위 메서드들 중 하나를 호출하면 `IllegalMonitorStateException`가 발생한다.

`wait()` 메서드를 호출하면 락을 해제하고, 스레드는 잠이 든다. 누군가 깨워줄 때 까지 `wait()`은 리턴되지 않는다. `notify()`, `notifyAll()` 메서드는 둘 다 `wait()`으로 잠든 메서드를 깨운다. 둘의 차이는 잠든 스레드 하나만 깨우냐, 모두 깨우냐의 차이이다. `notify()` 메서드는 어느 스레드를 깨울지 선택할 수 없기 때문에 제어가 어렵다. 그래서 보통은 `notifyAll()`을 사용한다. `notifyAll()`이 모든 스레드를 깨우긴 하지만 이 메서드를 호출한다고 해서 잠들어 있던 모든 스레드가 동시에 동작하는 것은 아니다. `wait()`으로 잠든 코드가 `synchronized` 블록 안에 있다는 것을 떠올려보자! `notifyAll()`로 깨어난 스레드들은 다시 락을 획득하기 위해 경쟁해야 한다. 락을 획득한 스레드만이 `wait()` 함수를 리턴시키고, 그 다음 로직을 수행할 수 있다.

## BlockingQueue 예제
그럼 이 내용을 바탕으로 `wait()`, `notifyAll()`을 이용하여 블로킹 큐를 구현해 보자. 이 큐는 다음과 같은 요구 사항을 갖고 있다.
* 생성 시점에 용량(capacity)이 결정된다.
* 큐가 비어있을 때 요소를 빼내려고 하면 빼낼 요소가 들어올 때까지 스레드가 블로킹된다.
* 용량이 꽉 찼을 때 요소를 추가하려고 하면 빈 공간이 생길 때까지 스레드가 블로킹된다.

아래 코드는 연결된 노드로 구현된 일반적인 큐이다. 이 코드를 조금 수정하여 블로킹 큐를 만들어 볼 것이다. 단순한 코드를 위해 큐의 요소는 문자열이라고 가정하였다.

```java
public class Queue {
  private int size;
  private Node head, tail;

  private static class Node {
    private String value;
    private Node next;
  }

  public boolean isEmpty() {
    return head == null;
  }

  public void enque(String item) {
    Node oldTail = tail;
    tail = new Node();
    tail.value = item;
    tail.next = null;
    if (isEmpty()) {
      head = tail;
    } else {
      oldTail.next = tail;
    }
    size++;
  }

  public String deque() {
    if (isEmpty()) {
      throw new NoSuchElementException();
    }
    String value = head.value;
    head = head.next;
    size--;
    if (isEmpty()) {
      tail = null;
    }
    return value;
  }
}
```

먼저 용량 제한을 위해 `capacity` 필드와 생성자를 추가하자.

```java
public class BlockingQueue {
  ...
  private final int capacity;

  public BlockingQueue(int capacity) {
    if (capacity <= 0) {
      throw new IllegalArgumentException();
    }
    this.capacity = capacity;
  }
  ...
}
```

큐가 다 찼는지 확인할 수 있도록 `isFull()` 메서드를 추가하자. `BlockingQueue`는 여러 스레드에서 접근하는 것을 가정하므로 스레드 안전해야 한다. 따라서 `isFull()` 메서드는 동기화해야 하고, 기존의 `isEmpty()` 메서드도 동기화해야 한다.

```java
  public synchronized boolean isFull() {
    return size == capacity;
  }

  public synchronized boolean isEmpty() {
    return head == null;
  }
```

이제 `enque()` 메서드를 수정해보자. 기존 구현은 용량 제한이 없었지만 이제는 용량이 꽉 찼을 때 스레드가 블로킹되도록(잠들도록) 해야한다. `wait()` 메서드를 이용하여 이를 구현할텐데, 앞서 설명했듯이 `wait()`을 호출하려면 고유 락을 먼저 획득해야 하므로 `enque()` 메서드에도 `synchronized`를 붙여준다.

```java
  public synchronized void enque(String item) {
    try {
      // 스레드가 깨어나도 큐가 꽉 찬 상태일 수 있으므로 if가 아닌 while로 구현한다.
      while (isFull()) {
        logger.debug("enque wait");
        wait();
        logger.debug("enque notified");
      }
    } catch (InterruptedException ex) {
    }

    // 나머지는 기존 구현과 동일

    notifyAll();
  }
```

마지막으로 `deque()`를 수정해 보자. 이 메서드는 큐가 비어있을 때 블로킹되어야 한다. 원리는 `enque()`와 동일하다.

```java
public synchronized String deque() {
  try {
    // 스레드가 깨어나도 큐가 꽉 비어있을 수 있으므로 if가 아닌 while로 구현한다.
    while (isEmpty()) {
      logger.debug("deque wait");
      wait();
      logger.debug("deque notified");
    }
  } catch (InterruptedException ex) {
  }

  T value = head.value;
  head = head.next;
  size--;
  if (isEmpty()) {
    tail = null;
  }
  notifyAll();
  return value;
}
```

이제 `BlockingQueue`를 사용하는 코드를 작성하여 테스트해보자.

```java
public static void main(String[] args) {
  BlockingQueue q = new BlockingQueue(1);
  Thread t = new Thread(() -> {
    try {
      Thread.sleep(200);
      q.enque("가");
      Thread.sleep(200);
      q.enque("나");
      q.enque("다");
    } catch (InterruptedException ex) {
    }
  });
  t.setName("work");
  t.start();

  try {
    logger.debug("{}", q.deque());
    Thread.sleep(1000); // 앞에서 200ms 대기하므로 실제로는 1200ms 후에 호출된다.
    logger.debug("{}", q.deque());
    logger.debug("{}", q.deque());
  } catch (InterruptedException ex) {
  }
}
```

테스트 편의를 위해 큐의 용량은 1로 제한하였다. 테스트에서는 메인 스레드와 워커 스레드가 하나씩 동작한다. 시간 순으로 일어나는 일을 나열하면 아래와 같다.

| 시간 | 스레드 | 메서드 호출 | wait/notify |
| --- | --- | --- | --- |
| 0ms | main | `deque()` | main wait |
| 200ms | work | `enque("가")` | main notified |
| 400ms | work | `enque("나")` | |
| | work | `enque("다")` | work wait |
| 1200ms | main | `deque()` | work notified |
| | main | `deque()` | &nbsp; |

결과는 아래와 같다.

```
22:46:55.267 [main] DEBUG BlockingQueue - deque wait
22:46:55.467 [main] DEBUG BlockingQueue - deque notified
22:46:55.467 [main] DEBUG BlockingQueue - 가
22:46:55.669 [work] DEBUG BlockingQueue - enque wait
22:46:56.471 [main] DEBUG BlockingQueue - 나
22:46:56.471 [work] DEBUG BlockingQueue - enque notified
22:46:56.472 [main] DEBUG BlockingQueue - 다
```

문제없이 예상한대로 동작하는 것을 확인하였다. 만약 실제로 이런 클래스가 필요하다면 만들어 쓰지말고 자바 표준 라이브러리에서 제공하는 [ArrayBlockingQueue][3]를 사용하도록 하자. 정확하고 성능이 좋은 동시성 프로그램은 직접 작성하기가 매우 어렵다. 그러므로 자바에서 제공하는 것을 사용하는 것이 프로젝트의 품질 뿐 아니라 개발자의 정신 건강에도 좋다.

## 참고 문서
[Java Object][1]

[1]: https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html
[2]: /2017/10/04/java-intrinsic-lock/
[3]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ArrayBlockingQueue.html
