---
title: CAP 이론과 PACELC 이론을 알아보자
date: 2017-07-29 20:40
categories:
- 이론
tags:
- CAP
- PACELC
---

이번에 CAP 이론을 공부하면서 가장 많이 참고한 자료는 [CAP Theorem: You don’t need CP, you don’t want AP, and you can’t have CA][3]라는 싯다르타 레디(Siddhartha Reddy)의 영상이다. 영어라 듣기가 조금 어려웠지만 내용 자체는 어렵지 않고, 강의 자료가 내용을 이해하는데 많은 도움이 되었다. 사실 이 글도 영상의 내용을 정리하고 부가 내용을 조금 더한 것에 지나지 않는다. 강의 자료는 [Speaker Deck][4]에 올라와 있으며 이 글 마지막에 첨부하였다.<!-- more -->

# CAP 이론
[CAP 이론][1]에서 CAP은 분산 데이터베이스 시스템의 세 가지 속성인 일관성(Consistency), 가용성(Availability), 파티션 허용성(Partition tolerance)을 나타낸다. 각각의 정의는 다음과 같다.

| 속성 | 설명 |
| --- | --- |
| **일관성** | 모든 요청은 최신 데이터 또는 에러를 응답받는다. |
| **가용성** | 모든 요청은 정상 응답을 받는다. |
| **파티션 허용성** | 노드간 통신이 실패하는 경우라도 시스템은 정상 동작 한다. |

여기서 말하는 일관성은 [ACID][2]의 일관성과는 다르다. ACID의 일관성은 데이터베이스에서 트랜잭션이 완료되었을 때 기존의 제약(무결성 등)을 위반하지 않아야 함을 의미한다. 만약 트랜잭션이 제약을 위반한다면 그 트랜잭션은 실패해야한다.

CAP 이론에 따르면 **분산 데이터베이스 시스템은 네트워크 파티션이 발생하였을 때 세 가지 속성 중 두 가지만 만족할 수 있다.** 세 가지 속성 중 두 가지를 만족하는 시스템은 CA(Consistency-Availability), CP(Consistency-Partition tolerance), AP(Availability-Partition tolerance) 세 종류로 나눌 수 있다. 하지만 CAP 이론은 네트워크 파티션 상황을 가정하므로 CA 시스템은 있을 수 없다. CA 시스템이 가능하려면 네트워크 파티션이 없어야 하는데 네트워크 파티션이 없으면 CAP 이론 자체가 쓸모 없을 뿐더러, 네트워크 파티션은 어떤 이유에서든 발생할 수 있다. 여기서 말하는 네트워크 파티션(Network Partition)은 물리적인 네트워크의 분할만을 의미하는 것이 아니다. 분산 시스템에서 노드끼리 데이터를 주고 받을 때 타임아웃이 발생하는 모든 경우를 네트워크 파티션으로 볼 수 있다. 타임아웃은 하드웨어의 문제 뿐만 아니라 방화벽 설정 오류, Java의 Garbage Collection 등 다양한 이유로 발생할 수 있다. 이런 이유로 우리는 네트워크 파티션이 없는 분산 시스템을 가정할 수 없으며, CAP 이론은 이러한 상황에서 분산 데이터베이스 시스템이 일관성(CP)과 가용성(AP) 둘 중에 하나만 만족할 수 있음을 설명한다.

# CAP 이론의 한계
CAP 이론은 분산 시스템의 성질을 너무나도 단순명료하게 설명해준다. 하지만 이 단순함때문에 CAP 이론은 한계를 갖는다. CAP 이론에 따르면 분산 시스템은 CP이거나 AP여야 한다. 하지만 실제 시스템은 둘 중에 하나라고 명확히 구분지을 수 없으며, 완벽한 CP시스템이나 완벽한 AP시스템은 사실상 쓸모가 없다. 하나씩 살펴보자.

* **완벽한 CP시스템** - 완벽한 일관성을 갖는 분산 시스템에서는 하나의 트랜잭션이 다른 모든 노드에 복제된 후에 완료될 수 있다. 이는 가용성 뿐만 아니라 성능의 희생을 필요로 한다. 이런 시스템은 하나의 노드라도 문제가 있으면 트랜잭션은 무조건 실패하고, 노드가 늘어날 수록 지연시간은 길어질 것이다.
* **완벽한 AP시스템** - 완벽한 가용성을 갖는 시스템은 모든 노드가 어떤 상황에서라도 응답할 수 있어야 한다. 하나의 노드가 네트워크 파티션으로 인해 고립되었다고 생각해보자. 이런 상황에서 고립된 노드가 갖고 있는 데이터는 쓸모가 없어지지만(일관성이 깨지므로) 어쨌든 응답한다면 완벽한 가용성을 갖게 된다. 운 나쁘게 이 노드와 연결된 사용자는 문제를 인지하지 못하고 계속해서 요청을 보낼 것이다.

따라서 더 나은 CAP 이론의 해석은 **'일관성과 가용성은 상충 관계에 있지만 둘 중에 반드시 하나만을 선택해야 하는 것은 아니다'** 이다. 완벽한 CP시스템과 완벽한 AP시스템 사이에는 수많은 가능성이 있다. 요구 사항에 따라 '다소 강한 일관성-다소 약한 가용성', '다소 약한 일관성-다소 강한 가용성'과 같이 일관성과 가용성의 수준을 선택해야 한다.
![일관성-가용성 스펙트럼](/images/cap-theorem-and-pacelc/cap.png)

CAP 이론의 또 다른 한계는 파티션이 없는 상황을 설명하지 못한다는 것이다. 파티션이 없는 상황에서도 분산 시스템은 상충하는 특성들이 있고, 장애 상황만큼이나 정상 상황에서 시스템이 어떻게 동작하는지도 중요하다.

# PACELC 이론
CAP 이론의 이러한 단점들을 보완하기 위해 나온 이론이 바로 [PACELC 이론][5]이다. CAP 이론이 네트워크 파티션 상황에서 일관성-가용성 축을 이용하여 시스템의 특성을 설명한다면, PACELC 이론은 거기에 정상 상황이라는 새로운 축을 더한다. PACELC는 P(네트워크 파티션)상황에서 A(가용성)과 C(일관성)의 상충 관계와 E(else, 정상)상황에서 L(지연 시간)과 C(일관성)의 상충 관계를 설명한다.
![PACELC](/images/cap-theorem-and-pacelc/pacelc.png)

y축 위쪽 끝이 지연시간이라고 표기되어서 오해를 사기 쉬운데, 위쪽에 위치할 수록 지연 시간이 길어지는 것이 아니라 짧아지는 것을 의미한다. 정상 상황에서 일관성에 치우친 시스템은 그만큼 지연 시간이 길어진다는 의미이다.
PACELC 이론에서는 장애 상황, 정상 상황에서 어떻게 동작하는지에 따라 시스템을 PC/EC, PC/EL, PA/EC, PA/EL로 나눌 수 있다. 강의에서는 잘 알려진 분산 데이터베이스 시스템들을 이 기준에 따라 나눠 본다. MySQL을 예로 들자면, 마스터-슬레이브로 구성된 MySQL 서버는 기본적으로 PA/EL이고, 설정에 따라 PA/EC가 될 수 있다. MySQL은 따로 설정하지 않으면 마스터에 트랜잭션 발생시 비동기적으로 슬레이브에 데이터를 복제(async replication)한다. MySQL에 semisynchronous replication 플러그인을 사용하는 경우에는 PA/EC 시스템이 되는데 이 플러그인은 다음과 같이 동작하기 때문이다.
* 마스터에서 트랜잭션 발생시 반드시 하나의 슬레이브로부터 복제 완료 알림 또는 타임아웃이 발생한다.(EC)
* 타임아웃이 발생하면 마스터는 비동기 복제(async replication)로 동작한다.(PA)

이외에 Zookeeper, Cassandra, Kafka도 같은 기준으로 설명하는데 자세한 내용은 강의 영상에서 확인할 수 있다.

# 선택의 기준
실제 시스템 개발에서 중요한 것은 어떤 종류의 시스템을 선택하여 사용할 지 정하는 것이다. PACELC가 대략적인 분류 기준을 제시하기는 하지만 같은 분류에 속하는 솔루션이라고 해서 완전히 똑같지는 않으며, 다양한 스펙트럼 상에 놓인 솔루션들을 비교하여 **비즈니스에 가장 적합한 솔루션** 을 사용해야 한다. 재고 관리 시스템에서는 일관성을 선택하는 것이 적합할 것이고, SNS에서는 가용성을 선택하는 것이 자연스러울 것이다.

# 관련 경험과 생각
예전에 프로젝트를 진행하면서 어느 데이터베이스 솔루션을 사용할 지 한참을 고민했던 적이 있다. 당시에 이 이론을 알았다면 솔루션을 선택하는데 도움이 됐을거란 생각이 든다. 하지만 PACELC 이론만 가지고 솔루션을 선택할 수는 없었을 것이다. 당시에 개발하던 시스템은 높은 수준의 성능이 요구됐었는데, PACELC 이론은 정상 상황에서의 성능을 단지 일관성과 연관지어 설명하지만 데이터베이스의 성능은 데이터의 생김새와 양과도 관련이 있기 때문이다. 데이터베이스 솔루션들은 PACELC와 관련된 특성들 뿐만 아니라 데이터의 생김새, 저장 방식에 따라 더 세분화된다. 저장할 데이터가 키-밸류 형태라면 키-밸류 스토리지에서 높은 성능을 기대할 수 있고, 데이터의 양이 많지 않다면 성능이 좋은 인메모리 데이터베이스를 사용할 수도 있다. 당시에 저장해야할 데이터는 양이 적었고, 키-밸류 형태였기 때문에 우리는 [Redis](https://redis.io/)를 사용하기로 결정했다.
그렇다고 해서 PACELC 이론을 몰라도 된다고 생각하는 것은 아니다. 데이터 생김새와 양은 일관성-가용성과는 다른 차원의 문제이기 때문이다. 일관성 혹은 가용성이 가장 중요한 선택 기준인 상황이거나, 일관성-가용성을 제외한 다른 조건이 모두 같다면 PACELC 이론은 적합한 솔루션을 선택할 수 있는 기준을 제시한다.

# 참고 문서
[CAP 이론(위키피디아)][1]
[ACID(위키피디아)][2]
[You don’t need CP, you don’t want AP, and you can’t have CA(영상)][3]
[You don’t need CP, you don’t want AP, and you can’t have CA(슬라이드)][4]
<script async class="speakerdeck-embed" data-id="41d0231fc29e4f9eac50b99f5edbc422" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>
[PACELC 이론(위키피디아)][5]

[1]: https://en.wikipedia.org/wiki/CAP_theorem
[2]: https://en.wikipedia.org/wiki/ACID
[3]: https://www.youtube.com/watch?v=hUd_9FENShA
[4]: https://speakerdeck.com/sids/cap-theorem-you-dont-need-cp-you-dont-want-ap-and-you-cant-have-ca
[5]: https://en.wikipedia.org/wiki/PACELC_theorem