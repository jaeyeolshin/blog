---
title: Log4j2 Filter의 onMatch, onMismatch 값들의 의미
date: 2018-07-04 22:00
categories:
- Log4j2
tags:
- log4j2
---

Log4j2의 Filter를 사용하면 다양한 방법을 로그를 제어할 수 있다. [Log4j2 - Filters][1] 문서를 보면 Log4j2가 제공하는 필터의 종류와 설정 방법이 설명되어 있다. 필터 종류마다 설정해야 하는 속성들이 조금씩 다르지만 모든 필터는 `onMatch`, `onMismatch` 속성을 갖고있다. 이 속성들은 각각 Filter에서 정의한 값과 매칭될 때 또는 그렇지 않을 때 동작을 의미한다. 여기에 들어갈 수 있는 값은 `ACCEPT`, `DENY`, `NEUTRAL` 세가지가 있다. 그런데 잠깐, `ACCEPT`는 로그를 쓰겠다는 의미일 것이고, `DENY`는 쓰지 않겠다는 의미일 것이다. 그럼 도대체 `NEUTRAL`은 뭘까?<!-- more -->

## 단일 Filter에서 NEUTRAL
단일 Filter에서는 `ACCEPT`와 `NEUTRAL`에 차이가 없다. 다시 말해, `DENY`만 아니면 필터를 통과하고 로그를 남긴다. 디버거에서 로그 로직을 따라가다 보면 다음과 같은 코드를 볼 수 있다. AbstractFilterable.java 코드의 일부이다.

```java
/**
 * Determine if the LogEvent should be processed or ignored.
 * @param event The LogEvent.
 * @return true if the LogEvent should be processed.
 */
@Override
public boolean isFiltered(final LogEvent event) {
    return filter != null && filter.filter(event) == Filter.Result.DENY;
}
```
(그러고보니 `@return` 주석이 잘못된 것 같다; `true if the LogEvent should be ignored.`가 맞는듯)
Filter의 결과가 `DENY`일 때만 로그 이벤트가 필터링되는 것을 확인할 수 있다. 실제로 그렇게 동작하는지 확인하기 위해 Logger나 Appender에 다음과 같이 MarkerFilter를 추가해보자.

```xml
<Logger name="myapp">
    <MarkerFilter marker="TEST" onMatch="ACCEPT" onMismatch="DENY"/>
    ...
</Logger>
```

MarkerFilter는 마커와 함께 쓰일 때 매치되므로 로그를 남길 때 Marker를 붙여야한다. 마커를 생성하여 로그 메서드 호출시 로그가 잘 남는지 확인해보자.

```java
package myapp;
// slf4j API를 사용해도 잘 동작한다.
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.Marker;
import org.slf4j.MarkerFactory;
...

public class App {
    private static final Logger LOG = LoggerFactory.getLogger("myapp");
    private static final Marker TEST = MarkerFactory.getMarker("TEST");

    public void doSomething() {
        ...
        LOG.debug(TEST, "로그0");
        ...
    }
}
```

로그가 잘 남았다면 `onMatch="NEUTRAL"`로 수정하여 다시 테스트 해보자. `ACCEPT`일 때와 동일하게 로그가 남을 것이다.

## CompositeFilter에서 NEUTRAL
`NEUTRAL`이 `ACCEPT`와 차이를 갖는 필터는 CompositeFilter이다. CompositeFilter는 여러 필터를 `<Filters>` 태그로 묶어서 정의한다. 앞서 정의했던 필터에 ThreadContextMapFilter를 추가하여 CompositeFilter를 정의해보자.

```xml
<Logger name="myapp">
    <Filters>
        <ThreadContextMapFilter onMatch="ACCEPT" onMismatch="DENY">
            <KeyValuePair key="debug" value="true"/>
        </ThreadContextMapFilter>
        <MarkerFilter marker="TEST" onMatch="ACCEPT" onMismatch="DENY"/>
    </Filters>
    ...
</Logger>
```

ThreadContextMapFilter는 [ThreadContext][2]에 있는 키/값 쌍이 일치할 때만 매치된다. ThreadContext는 [ThreadLocal](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html) 기반으로 동작하므로 웹앱에서 특정 요청의 로그를 남기는데 사용할 수 있다.

위와 같이 CompositeFilter를 구성하면 ThreadContextMapFilter와 MarkerFilter에서 둘 다 ACCEPT되면 로그가 남을 것 같다. 하지만 실제로는 그렇게 동작하지 않는다. 코드를 수정하여 테스트 해보자.

```java
public class App {
    private static final Logger LOG = LoggerFactory.getLogger("myapp");
    private static final Marker TEST = MarkerFactory.getMarker("TEST");

    public void doSomething() {
        ...
        ThreadContext.put("debug", "true");
        LOG.debug(TEST, "로그0");
        LOG.debug("로그1");
        ThreadContext.remove("debug");
        LOG.debug("로그2");
        ...
    }
}
```

예상한대로라면 "로그0"만 남아야한다. "로그1"은 마커가 빠졌고, "로그2"는 ThreadContext 값을 지우고 난 뒤에 호출되었기 때문이다. 하지만 실제 출력된 로그는 다음과 같다.

```bash
로그0
로그1 # 마커가 없는데도 출력되었다.
```

당황스럽지만 원인을 찾기 위해 CompositeFilter 코드를 찾아 들어가보자. CompositeFilter의 `filter()`메서드 구현은 아래와 같다.
```java
@Override
public Result filter(final Logger logger, final Level level, final Marker marker, final String msg,
        final Object... params) {
    Result result = Result.NEUTRAL;
    for (int i = 0; i < filters.length; i++) {
        result = filters[i].filter(logger, level, marker, msg, params);
        if (result == Result.ACCEPT || result == Result.DENY) {
            return result;
        }
    }
    return result;
}
```

정의된 필터를 순서대로 실행하되 그 결과가 `ACCEPT`나 `DENY`면 바로 그 값을 리턴해버린다. 그래서 "로그1"을 쓰려고 할 때 MarkerFilter가 무시된 것이다. 여기서는 `NEUTRAL`을 써야한다. 설정을 아래와 같이 바꾸면 우리가 기대한대로 동작한다.

```xml
<Logger name="myapp">
    <Filters>
        <ThreadContextMapFilter onMatch="NEUTRAL" onMismatch="DENY">
            <KeyValuePair key="debug" value="true"/>
        </ThreadContextMapFilter>
        <!-- 마지막 필터의 onMatch는 ACCEPT로 설정해도 상관없다 -->
        <MarkerFilter marker="TEST" onMatch="NEUTRAL" onMismatch="DENY"/>
    </Filters>
    ...
</Logger>
```

다시 테스트해보면 "로그0"만 남는 것을 확인할 수 있다.

> ACCEPT도 DENY도 아닌 중립적인 동작이라니...

log4j2 개발자자들은 더 미세한 로그 필터링을 위해 `ACCEPT`, `DENY`, `NEUTRAL` 세 개의 값을 도입하였을 것이다. 하지만 개발을 하다보면 '이것 아니면 저것'이라는 이분법적 사고에 익숙해지기 때문에 `NEUTRAL`이라는 값은 개발자들에게 혼란을 주는 것 같다. 어쨌든 이제는 그 의미를 알았으니 자유롭게 log4j2의 Filter 기능을 활용해보자!

## 참고 문서
[Log4j2 - Filters][1]
[Log4j2 - ThreadContext][2]


[1]: https://logging.apache.org/log4j/2.0/manual/filters.html
[2]: https://logging.apache.org/log4j/2.x/manual/thread-context.html
