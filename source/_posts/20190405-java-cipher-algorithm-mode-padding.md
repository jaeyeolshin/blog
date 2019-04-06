---
title: Java Cipher - 알고리즘, 운용 모드, 패딩의 이해
date: 2019-04-06 00:30
categories:
- Java
- Encryption
- Security
tags:
- java
- encryption
- security
---

자바에서는 대칭키 알고리즘을 사용하여 데이터를 암호화/복호화할 때 [javax.crypto.Cipher][1] 클래스를 사용한다. 이 클래스의 인스턴스는 정적 메서드인 `Cipher.getInstance()`를 호출하여 가져올 수 있는데, 호출시 사용할 알고리즘, 운용 모드, 패딩 방식을 인자로 넘겨줘야 한다. 대칭키 암호화에서 알고리즘, 운용 모드, 패딩은 무엇이고 어떤 역할을 하는지 알아보자.<!-- more -->

## 혼돈과 확산
알고리즘 등에 대해 얘기하기전에 암호문의 성질에 대해서 알아보자. 안전한 암호문은 공격자가 이를 보고 원본 메시지나 암호화에 사용된 키를 유추할 수 없어야한다. 이러한 성질을 혼돈과 확산이라고 하며, 각각 다음과 같은 의미를 갖는다.

* 혼돈(confusion)은 암호문으로부터 키를 알아낼 수 없게 하는 성질이다.
* 확산(diffusion)은 암호문으로부터 원문을 알아낼 수 없게 하는 성질이다.

다르게 말하면 혼돈이란 키의 비트 하나만 바꿔도 암호문 전체가 바뀌도록 하는 성질이다. 안전한 암호는 공격자가 아무리 많은 평문-암호문 쌍을 알고 있어도 그 속에서 키의 패턴을 발견할 수 없어야 한다. 마찬가지로 확산은 원문의 비트를 하나만 바꿔도 암호문 전체가 바뀌도록 하는 성질이다. 서로 다른 원문이 비슷한 내용을 담고 있더라도 각각의 암호문은 완전히 다른 값을 가져야 한다.

## 암호 알고리즘
암호 알고리즘에서는 혼돈과 확산을 달성하기 위해 Substitution과 Permutation을 이용한다. Substitution은 문자를 다른 문자로 바꾸는 것이고, Permutation은 문자들의 순서를 바꾸는 것이다. Substitution과 Permutation을 한번 수행하는 것이 암호 알고리즘의 기본 수행 단위이다. 암호문은 이를 여러번 수행할 수록 안전하다. Substitution-Permutation을 연속하여 수행하도록 이어 놓은 것을 SPN(Substitution Permutation Network)이라고 한다. SPN에서 한 번의 Substitution-Permutation 수행을 라운드 또는 레이어라고 한다. 아래 그림은 3라운드로 이루어진 SPN이다.(이미지 출처: [Wikipedia][2])

<img src="/images/java-cipher-algorithm-mode-padding/SubstitutionPermutationNetwork2.png" width="360" alt="Substitution Permutation Network">

SPN을 이용하는 대표적인 암호 알고리즘으로 AES가 있다. SPN을 이용하는 알고리즘은 보통 데이터를 블록 단위로 처리한다. AES의 경우 블록의 크기는 128비트(16바이트)이다. 그런데 모든 데이터가 16바이트 크기를 가질 수 없으므로, 데이터를 블록 단위로 나누어 처리하고 합치는 과정이 필요하다.

## 운용 모드
데이터를 블록으로 나누어 처리하고 합치는 것, 그것이 암호화에서 운용 모드가 하는 역할이다. 대표적인 운용 모드로는 ECB(Electronic Code Book)와 CBC(Cipher Block Chaining)가 있다.

먼저 ECB부터 알아보자. ECB는 단순히 블록 단위로 처리한 결과를 이어붙이는 방법이다. 아래 그림은 ECB 처리 방식을 보여준다.(이미지 출처: [Wikipedia][3])

![Electronic Code Book](/images/java-cipher-algorithm-mode-padding/1202px-ECB_encryption.svg.png)

ECB는 단순하지만 같은 값을 갖는 원문 블록은 같은 암호 블록을 출력하기 때문에 원문의 패턴이 그대로 드러난다. 이는 암호문에서 원문을 유추할 수 있음을 의미하기 때문에 ECB는 확산의 성질을 달성하지 못한다고 볼 수 있다. 아래는 원본 이미지와 ECB로 암호화한 이미지이다. 원본의 패턴이 드러나는 것을 볼 수 있다.(이미지 출처: [Wikipedia][3])

<div style="display: flex; justify-content: center;">
  <div>
    <img src="/images/java-cipher-algorithm-mode-padding/Tux.jpg" alt="원본">
  </div>
  <div>
    <img src="/images/java-cipher-algorithm-mode-padding/Tux_ecb.jpg" alt="ECB">
  </div>
  <div>
    <img src="/images/java-cipher-algorithm-mode-padding/Tux_secure.jpg" alt="안전한 모드">
  </div>
</div>

CBC는 원문 블록을 그대로 암호화하지 않고, 직전에 암호화된 블록과 XOR 연산을 한 다음에 암호화를 수행한다. 그렇기 때문에 같은 내용을 갖는 원문 블록이라도 전혀 다른 암호문을 갖게 된다. 그런데 첫번째 블록은 직전 암호문이 없으므로 XOR 연산 대상이 없다. 이를 위해 CBC 모드는 초기화 벡터(initialization vector)를 입력받는다. 초기화 벡터는 원문 블록을 XOR하는데 쓰이기 때문에 당연히 블록 사이즈와 동일한 크기여야 한다. 아래는 CBC 방식을 나타낸 그림이다.(이미지 출처: [Wikipedia][3])

![Cipher Block Chaining](/images/java-cipher-algorithm-mode-padding/1202px-CBC_encryption.svg.png)

CBC 모드는 직전 블록이 다음 블록의 암호화에 관여하므로 일부 블록만 복호화하고 싶어도 전체를 복호화해야 한다. 반면 ECB는 전체를 복호화하지 않고 일부만 복호화하는 것이 가능하다.

## 패딩
AES와 같은 16바이트 크기의 블록 암호 알고리즘을 사용하는데 원문의 크기가 16바이트의 배수가 아니라면 마지막 블록은 16바이트보다 작은 크기가 된다. 이 때 마지막 블록의 빈 부분을 채워주는 방식을 패딩이라고 한다. 가장 유명한 패딩 방식인 PKCS5과 PKCS7을 알아보자.
PKCS5는 8바이트 블록의 암호 알고리즘을 가정한다. 원문의 길이가 `L` 바이트이면 마지막 블록은 `L mod 8`의 크기를 갖는다. 그럼 패딩 크기는 `8 - (L mod 8)`가 된다. PKCS5는 단순히 패딩 크기의 값을 갖는 바이트를 크기만큼 반복한다. 아래 표는 패딩 크기와 실제 패딩으로 채워지는 데이터를 나타낸다.

| 8 - (L mod 8) | 패딩 바이트 |
| --- | --- |
| 1 | 01 |
| 2 | 02 02 |
| ... | ... |
| 8 | 08 08 08 08 08 08 08 08 |

근데 8바이트일 때 즉, 원문이 블록 크기로 나누어 떨어질 때도 패딩이 들어가는 것을 볼 수 있다. 왠지 나누어 떨어지면 패딩을 넣지 않아도 될 것 같지만 그렇지 않다. 블록 크기로 나누어 떨어지는 원문에 패딩을 넣지 않는다고 가정해보자. 이 때 원문의 마지막 바이트가 `01`이라면 이게 패딩인지 실제 데이터인지 구분할 방법이 없다. 그렇기 때문에 패딩을 할 때는 일관되게 패딩 바이트를 추가해주는 것이다.

PKCS7은 8바이트가 아닌 가변 길이를 갖는 다는 점에서 다르지만 원리는 PKCS5와 동일하다. PKCS7에서는 블록 크기가 1에서 255까지의 값을 가질 수 있다.(255는 한 바이트가 가지는 가장 큰 값이다.) 자바에서는 패딩 방식을 입력할 때 PKCS5와 PKCS7를 구분하지 않고 `PKCS5Padding` 이라고 입력한다.

## 자바 코드 예제
그럼 지금까지 이해한 내용을 자바 코드로 작성해보자.

### AES/ECB
우선 AES 알고리즘을 ECB 운용 모드와 함께 사용하는 코드를 보자. 패딩 방식은 바꿔가면서 결과를 확인해볼 것이다.

```java
public static void main(String[] args) throws Exception {
  Cipher cipher = Cipher.getInstance("AES/ECB/NoPadding");
  // Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
  // key를 생성하기 위해 SHA1를 사용한다
  byte[] hash = MessageDigest.getInstance("SHA1")
      .digest("myKey".getBytes(StandardCharsets.US_ASCII));
  // 128bit 키를 만들기위해 16바이트만큼 자른다
  byte[] keyBytes = Arrays.copyOf(hash, 16);
  Key key = new SecretKeySpec(keyBytes, "AES");
  cipher.init(Cipher.ENCRYPT_MODE, key);

  byte[] src = "0123456789abcdef".getBytes(StandardCharsets.US_ASCII);
  byte[] enc = cipher.doFinal(src);
  // 16진수 출력을 위해 org.apache.commons.codec.binary.Hex를 사용한다
  System.out.println(Hex.encodeHexString(enc));
}
```
```bash
# 입력: 0123456789abcdef
# 알고리즘: AES/ECB/NoPadding
56340d042f8fc5f91ace63588569042d
```
패딩이 없기 때문에 입력 크기를 16바이트 배수로 맞춰줘야한다. 크기가 16바이트 배수가 아니면 `javax.crypto.IllegalBlockSizeException`가 발생한다. 입력이 16바이트기 때문에 패딩을 `PKCS5Padding`로 바꾸면 암호문에 16바이트만큼 패딩이 추가된다.
```bash
# 입력: 0123456789abcdef
# 알고리즘: AES/ECB/PKCS5Padding
56340d042f8fc5f91ace63588569042d3ee243e223a71541cc3bf48118b4c37f
```

다시 NoPadding으로 바꾸고 입력을 복사해서 크기를 두배로 늘리면 패턴이 반복되는 것을 볼 수 있다.
```bash
# 입력: 0123456789abcdef0123456789abcdef
# 알고리즘: AES/ECB/NoPadding
56340d042f8fc5f91ace63588569042d56340d042f8fc5f91ace63588569042d
```

### AES/CBC
그다음으로 AES와 CBC 운용 모드를 사용해보자. CBC는 암호화 전에 XOR 연산을 하므로 초기화 벡터가 필요하다고 했다. 또한 ECB와 달리 원문 블록의 패턴이 반복되더라도 암호문에서는 그 패턴이 보이지 않기 때문에 ECB보다 더 안전한 방식이라고 했다. 실제로 그런지 확인해보자.

```java
public static void main(String[] args) throws Exception {
  Cipher cipher = Cipher.getInstance("AES/CBC/NoPadding");
  // key를 생성하기 위해 SHA1를 사용한다.
  byte[] keyHash = MessageDigest.getInstance("SHA1")
      .digest("myKey".getBytes(StandardCharsets.US_ASCII));
  // 128bit 키를 만들기위해 16바이트만큼 자른다.
  byte[] keyBytes = Arrays.copyOf(keyHash, 16);
  Key key = new SecretKeySpec(keyBytes, "AES");

  byte[] ivHash = MessageDigest.getInstance("SHA1")
      .digest("myIv".getBytes(StandardCharsets.US_ASCII));
  byte[] ivBytes = Arrays.copyOf(ivHash, 16);
  IvParameterSpec iv = new IvParameterSpec(ivBytes);
  cipher.init(Cipher.ENCRYPT_MODE, key, iv);

  byte[] src = "0123456789abcdef".getBytes(StandardCharsets.US_ASCII);
  byte[] enc = cipher.doFinal(src);
  System.out.println(Hex.encodeHexString(enc));
}
```
```bash
# 입력: 0123456789abcdef
# 알고리즘: AES/ECB/NoPadding
1a094d8661c12b93e631af952892ee1c
```
만약 `Cipher.init()` 호출시 초기화 벡터를 넣어주지 않는다면 임의로 생성된 초기화 벡터가 사용된다. 초기화 벡터는 복호화할 때도 필요하기 때문에 `Cipher.getIV()`를 호출하여 임의 생성된 값을 얻어와야한다.

그럼 ECB에서 했던 것 처럼 입력을 복사해서, 패턴이 반복되는지 확인해보자.
```bash
# 입력: 0123456789abcdef0123456789abcdef
# 알고리즘: AES/CBC/NoPadding
1a094d8661c12b93e631af952892ee1cac84b15151a0845539203d784dea1f3b
```
ECB와 달리 암호문이 반복되지 않는 걸 확인할 수 있다.

## 참고 문서
[인크립션 - 실용주의 암호화](http://www.yes24.com/Product/Goods/32439157)
[Java Cipher][1]
[Wikepedia - Substitution permutation network][2]
[Wikepedia - Block cipher mode of operation][3]

[1]: https://docs.oracle.com/javase/8/docs/api/javax/crypto/Cipher.html
[2]: https://en.wikipedia.org/wiki/Substitution%E2%80%93permutation_network
[3]: https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation
