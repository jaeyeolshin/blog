---
title: Java InterruptedException은 어따 쓰는겨?
date: 2017-10-09 22:30
categories:
- Java
tags:
- java
- concurrency
- thread
- interrupt
---

## Thread.sleep()과 InterruptedException
자바 개발자라면 [Thread.sleep()][1] 메서드를 한번쯤은 써봤을 것이다. 이 메서드는 제품 단계의 코드에서는 잘 쓰이지 않지만 테스트 코드 등에서 어떤 로직이 실행될 때까지 특정 시간동안 기다려야 할 때 사용된다. 그런데 이 메서드를 사용하려고 할 때마다 컴파일러가 불평을 늘어놓는다. 그저 코드 실행을 몇 초 뒤로 미루고 싶었을 뿐인데...
<!-- more -->

![Thread.sleep()에 대한 컴파일러의 불평](/images/java-thread-interrupt/unhandled_interruptexception.png)

보통 우리는 이런 컴파일러의 불평을 잠재우기 위해 `try-catch` 문으로 감싸주고, `catch` 블록에서 잡힌 [InterruptedException][2]은 무시한다. 매번 `try-catch` 문을 쓰기 싫어하는 개발자는 아래와 같이 예외를 무시하는 헬퍼 메서드를 만들어 사용할 수도 있다. 하지만 무작정 예외를 무시하는 것은 좋지 않은 방법이다. 이 예외는 항상 적절히 처리되어야 한다.

```java
public static void sleep(long millis) {
  try {
    Thread.sleep(millis);
  } catch (InterruptedException ex) {
    // 간단한 테스팅 코드나 장난감 코드가 아니라면 이렇게 하지 말자!
    // InterruptedException은 적절히 처리되어야 한다.
  }
}
```

그럼 어떻게 하는 것이 `InterruptedException`을 적절히 처리하는 것일까? 우선 인터럽트가 무엇인지부터 알아보자.

## Java Thread의 인터럽트(interrupt) 메커니즘
`InterruptedException`은 자바 스레드의 인터럽트 메커니즘의 일부이다. 자바에서는 스레드에 하던 일을 멈추라는 신호를 보내기 위해 인터럽트를 사용한다. 한 스레드가 다른 스레드를 인터럽트 할 수 있고, 각 스레드는 자신이 인터럽트 되었는지 확인할 수 있다. 스레드가 자기 자신을 인터럽트 할 수도 있다. 인터럽트된 스레드에서 이를 어떻게 처리해야 한다는 규칙은 없다. 하지만 대부분의 경우 인터럽트는 하던 일을 멈추라는 신호이며, 해당 스레드는 이를 적절히 처리해야 한다.

어떤 스레드를 인터럽트 하고 싶으면 대상 스레드의 [Thread.interrupt()][3] 메서드를 호출하면 된다. 그러면 대상 스레드에서는 아래 중 하나의 일이 일어나게 된다.

* 현재 스레드가 대상 스레드를 수정할 수 있는 권한이 없으면 [SecurityException][4]이 발생한다.
* 대상 스레드가 `Object.wait()`, `Thread.sleep()`, `Thread.join()`, `Future.get(), BlockingQueue.take()` 등의 메서드에 의해 블로킹된 경우, **interrupt state가 사라지고** `InterruptedException`이 발생한다.
* 대상 스레드가 [InterruptibleChannel][5]을 이용한 I/O 작업에 의해 블로킹된 경우, **interrupt state가 설정되고** [ClosedByInterruptException][6]이 발생한다.
* 대상 스레드가 [Selector][7]에서 블로킹된 경우, **interrupt state가 설정되고** selection 작업에서 리턴된다.
* 이외의 경우에는 **interrupt state가 설정된다.**

interrupt state는 대상 스레드가 자신이 인터럽트 되었는지 확인할 수 있는 상태 값이라고 보면 된다. 상태 값은 [Thread.interrupted()][8]와 [Thread.isInterrupted()][9] 두 메서드로 확인할 수 있다. 첫 번째 메서드는 정적 메서드이고, 호출 후에 interrupt state가 사라진다. 두 번째 메서드는 인스턴스 메서드이며, 호출 후에도 interrupt state는 유지된다. 요약하자면 스레드는 자신이 인터럽트 되었는지 두 가지 방법으로 확인할 수 있다.
1. interrupt state가 설정되었는지 확인
2. `InterruptedException`이 발생했는지 확인

그렇다면 인터럽트가 발생했을 때 해당 스레드는 이를 어떻게 처리해야 할까?

## 인터럽트 처리 - 스레드 풀을 사용하는 경우
Java 5부터 [ExecutorService][10](이하 실행자)가 추가되면서 직접 스레드를 생성할 필요가 없어졌다. [Executors][12]에서 지원하는 팩터리 메서드들을 통해 필요한 스레드 풀을 생성하여 사용할 수 있다.
무한 루프를 도는 작업을 스레드풀에서 실행해야한다고 가정해보자. 아래와 같이 `Runnable` 구현체를 만들어 스레드풀에서 수행하도록 하면 어떻게 될까?

```java
public class LoopTask implements Runnable {
  @Override
  public void run() {
    while (true) {
      // do something
    }
  }
}
```

이렇게 해도 루프 안의 로직들은 잘 실행되겠지만 `LoopTask`는 정상적인 방법으로 종료될 수 없다. 스레드풀 내의 작업을 종료 시키기 위해서 [Future.cancel()][12] 혹은 [Executor.shutdownNow()][13]을 사용하는데 이 메서드들은 스레드의 인터럽트 메커니즘에 의존한다. 종료시키려는 스레드가 인터럽트에 반응하지 않는다면 작업을 종료할 수 없다.

```java
Executor exec = Executors.newSingleThreadExecutor();
Future<?> f = exec.submit(new LoopTask());

f.cancel(true); // 또는 exec.shutdownNow()를 호출해도 종료되지 않는다.
```

따라서 스레드가 인터럽트에 적절히 처리될 수 있도록 `LoopTask`를 아래와 같이 수정해야 한다.

```java
public class LoopTask implements Runnable {
  @Override
  public void run() {
    while (!Thread.currentThread().isInterrupted()) {
      // do something
    }
  }
}
```

이제 `LoopTask`는 루프를 돌 때마다 인터럽트 상태를 읽어서 인터럽트되었으면 작업을 종료한다. 만약 `LoopTask`가 `InterruptedException`을 던지는 메서드를 사용한다면 아래와 같이 작성되어야 한다.

```java
public class LoopTask implements Runnable {
  @Override
  public void run() {
    while (!Thread.currentThread().isInterrupted()) {
      try {
        // do something
        Thread.sleep(100);
      } catch (InterruptedException ex) {
        Thread.currentThread().interrupt();
      }
    }
  }
}
```

`InterruptedException`이 발생할 때는 스레드의 interrupt state가 설정되지 않는다. 따라서 예외 처리문에서 직접 interrupt state를 설정하여 작업이 종료되게 해야한다. 여기서 `Thread.currentThread().interrupt()`를 호출하는 대신 그냥 `break`로 빠져나가면 안되냐고 생각할 수 있다. 그렇게 하면 작업은 종료되지만 스레드를 관리하는 상위 레이어(여기서는 스레드풀)에 인터럽트 발생 여부를 알릴 수 없게 된다. 스레드풀 역시 발생한 인터럽트를 적절히 처리할 수 있도록 interrupt state를 설정해야 한다.

## 인터럽트 처리 - 직접 스레드를 생성하는 경우
스레드풀을 사용하는 것과 달리 스레드를 직접 생성하여 사용하면 스레드를 직접 참조할 수 있다. 인터럽트를 처리할 때 스레드를 관리하는 상위 레이어에 인터럽트 상태를 알릴 필요도 없기 때문에 굳이 `InterruptedException` 발생시 interrupt state를 설정하지 않아도 된다. 굳이 인터럽트 메커니즘을 사용하지 않고 플래그 변수로 루프를 종료하도록 해도 된다. 하지만 개인적으로는 습관처럼 루프 종료 조건에 interrupt state 확인을 추가하고, `InterruptedException` 발생시 interrupt state를 설정하는 것이 좋은 것 같다. 이렇게 하면 스레드를 직접 생성하여 처리하던 로직을 스레드풀에서 수행하도록 변경해도 문제가 없다.

```java
public class LoopTaskThread extends Thread {
  ...
  @Override
  public void run() {
    while (!Thread.currentThread().isInterrupted()) {
      try {
        // do something
        Thread.sleep(100);
      } catch (InterruptedException ex) {
        this.interrupt();
      }
    }
  }
  ...
}

Thread t = new LoopTaskThread();
t.start();
// 종료해야 할 때
t.interrupt();
```

## 요약
* 인터럽트는 스레드를 종료하기 위한 메커니즘이다.
* 테스트 코드나 장난감 코드가 아니라면 인터럽트를 적절히 처리하도록 하자.
* 멀티스레드로 동작하는 코드에 무한 루프 혹은 `InterruptedException`을 던지는 메서드 호출이 존재한다면, 아래 코드를 관용구처럼 사용하자.

```java
while (!Thread.currentThread().isInterrupted()) {
  try {
    // Thread.sleep(), Future.get(),
    // BlockingQueue.take() 등이 여기올 수 있다.
  } catch (InterruptedException ex) {
    Thread.currentThread().interrupt();
  }
}
```

## 참고 문서
[Java Thread.interrupt][3]

[1]: https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#sleep-long-
[2]: https://docs.oracle.com/javase/8/docs/api/java/lang/InterruptedException.html
[3]: https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#interrupt--
[4]: https://docs.oracle.com/javase/8/docs/api/java/lang/SecurityException.html
[5]: https://docs.oracle.com/javase/8/docs/api/java/nio/channels/InterruptibleChannel.html
[6]: https://docs.oracle.com/javase/8/docs/api/java/nio/channels/ClosedByInterruptException.html
[7]: https://docs.oracle.com/javase/8/docs/api/java/nio/channels/Selector.html
[8]: https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#interrupted--
[9]: https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#isInterrupted--
[10]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html
[11]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html
[12]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html#cancel-boolean-
[13]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html#shutdownNow--
