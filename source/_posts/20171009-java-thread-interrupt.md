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

보통 우리는 이런 컴파일러의 불평을 잠재우기 위해 `try-catch` 문으로 감싸주고, `catch` 블록에서 잡힌 [InterruptedException][2]은 무시한다. 영리한 개발자라면 `try-catch` 문을 반복해서 사용하는 대신 아래와 같은 헬퍼 메서드를 작성할 것이다.

```java
public static void sleep(long millis) {
  try {
    Thread.sleep(millis);
  } catch (InterruptedException ex) {
  }
}
```

어쨌든 우리는 몇 초간 기다리는 코드를 위해 `InterruptedException`을 처리해야 한다. 개발자들을 귀찮게 하는 `InterruptedException`. 원래의 용도는 대체 무엇이며, 왜 `Thread.sleep()`은 이 예외를 던지도록 설계되었을까?

## Java Thread의 인터럽트(interrupt) 메커니즘
`InterruptedException`은 자바 스레드의 인터럽트 메커니즘의 일부이다. 자바에서는 스레드에 신호를 보내기 위해 인터럽트를 사용한다. 한 스레드가 다른 스레드를 인터럽트 할 수 있고, 각 스레드는 자신이 인터럽트 되었는지 확인할 수 있다. 스레드가 자기 자신을 인터럽트 할 수도 있다. 인터럽트를 받은 스레드에서 이 신호를 반드시 어떻게 처리해야 한다는 규칙은 없다. 개발자가 비즈니스 로직에 맞게 처리해주면 된다. 위에서 작성했던 헬퍼 메서드 `sleep()`은 신호를 무시하도록 작성되었는데 이는 의도된 것이므로 문제 없다.

어떤 스레드를 인터럽트 하고 싶으면 대상 스레드의 [Thread.interrupt()][3] 메서드를 호출해주면 된다. 그러면 대상 스레드에서는 아래 중 하나의 일이 일어나게 된다.

* 현재 스레드가 대상 스레드를 수정할 수 있는 권한이 없으면 [SecurityException][4]이 발생한다.
* 대상 스레드가 `Object.wait()`, `Thread.sleep()`, `Thread.join()` 메서드에 의해 블로킹된 경우, **interrupt state가 사라지고** `InterruptedException`이 발생한다.
* 대상 스레드가 [InterruptibleChannel][5]을 이용한 I/O 작업에 의해 블로킹된 경우, **interrupt state가 설정되고** [ClosedByInterruptException][6]이 발생한다.
* 대상 스레드가 [Selector][7]에서 블로킹된 경우, **interrupt state가 설정되고** selection 작업에서 리턴된다.
* 이외의 경우에는 **interrupt state가 설정된다.**

interrupt state는 대상 스레드가 자신이 인터럽트 되었는지 확인할 수 있는 상태 값이라고 보면 된다. 상태 값은 [Thread.interrupted()][8]와 [Thread.isInterrupted()][9] 두 메서드로 확인할 수 있다. 첫 번째 메서드는 정적 메서드이고, 호출 후에 interrupt state가 사라진다. 두 번째 메서드는 인스턴스 메서드이며, 호출 후에도 interrupt state는 유지된다. 요약하자면 스레드는 자신이 인터럽트 되었는지 두 가지 방법으로 확인할 수 있다.
1. interrupt state가 설정되었는지 확인
2. `InterruptedException`이 발생했는지 확인

## 인터럽트 사용 예시
이제 `Thread.sleep()`이 왜 `InterruptedException`을 던지도록 설계되었는지는 알았을 것이다. 그럼 상황을 가정해 예제 코드를 한 번 만들어 보자. 자바로 작성된 비디오 인코딩 프로그램이 있다. 이 프로그램에 원본 파일을 넣고, 타겟 포맷을 설정한 후 실행 버튼을 누르면 인코딩이 시작된다. 사용자는 언제든지 취소 버튼을 누를 수 있다. 실행 버튼을 눌렀을 때 여러 스레드가 생성되어 함께 작업을 처리하는데 그 중 하나는 1초에 한 번 진행 상황을 알려주는 스레드이다. 이 스레드는 인코딩이 완료되거나, 사용자가 취소 버튼을 눌렀을 때 종료되어야 한다. 종료는 인터럽트 메커니즘을 사용한다.

```java
public class NotifyingThread extends Thread {
  ...
  @Override
  public void run() {
    while (!Thread.interrupted()) {
      try {
        Thread.sleep(1000);
      } catch (InterruptedException ex) {
        this.interrupt();
        continue;
      }
      // 진행 상황을 파악하여 사용자 UI 업데이트.
    }
  }
  ...
}
```

이 스레드는 열심히 일을 하다가도 인터럽트가 발생하면 종료된다. `catch` 블록에서 그냥 `return`을 해도 되고, 위와 같이 interrupt state를 설정해주고, `while` 문에서 종료되도록 해도 된다.

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
