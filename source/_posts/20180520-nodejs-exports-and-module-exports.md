---
title: Node.js에서 exports와 mudule.exports의 차이
date: 2018-05-20 17:00
categories:
- Node.js
tags:
- nodejs
---

요 며칠간 Node.js를 공부하고 있다. 회사에서 주로 자바로 개발하고 있고, 개인적으로도 자바와 같은 객체 지향 + 정적 타입 언어가 익숙해서 좋아하지만 사람 일은 모르는 것 아니겠는가. 어딜 갖다놔도 쓸만한 사람이 되려면 아무쪼록 이것저것 배우는 게 좋을 것 같아서 다른 언어도 배우고 있다. [Mozilla에 잘 정리된 Express/Node.js 튜토리얼][1]이 있길래 문서를 차근차근 읽어보는 중이다.(Mozilla 사이트의 문서들은 하나같이 깔끔하고 좋은 것 같다!)<!-- more -->

## exports
Node.js에서는 분리된 소스 파일을 별도의 모듈처럼 사용할 수 있다. 모듈을 생성하려면 소스 파일(.js)을 만들고 모듈화할 변수나 함수를 `exports` 객체의 프로퍼티로 할당하면 된다. 예제로 정사각형의 넓이와 둘레를 구하는 함수를 모듈로 만들어보자.

우선 모듈명에 맞는 소스파일을 생성한다. 정사각형과 관련된 모듈이므로 square.js라는 파일을 생성하고, `area()`, `perimeter()` 함수를 exports 하자.

```javascript
// square.js
exports.area = (width) => width * width;
exports.perimeter = (width) => 4 * width;
```

이렇게 만든 `square` 모듈은 다른 코드에서 사용할 수 있다. 모듈을 사용하려면 `require()` 메서드로 모듈 객체를 임포트해야 한다.

```javascript
// app.js
const square = require('./square'); // 앱이 실행되는 상대 경로를 입력해도 되고, 절대 경로를 입력해도 된다.
console.log(square.area(3));
console.log(square.perimeter(3));
```

앱을 실행하면 예상했던 결과가 출력된다.
```base
$ node app.js
9
12
```

## module.exports
기능을 모듈화할 때 사용할 수 있는 객체가 하나 더 있다. 바로 `module.exports`이다. [튜토리얼의 Importing and creating modules][2]를 보면 `module.exports`는 개별 함수가 아닌 객체를 통째로 exports할 때 사용하라고 나와있다. 위의 예제와 똑같은 `square` 모듈을 만들기 위해 다음과 같이 해도된다.

```javascript
// square.js
module.exports = {
  area: (width) => width * width,
  perimeter: (width) => 4 * width
}
```

이렇게 해도 같은 결과가 나오는 것을 확인할 수 있다.

## exports, module.exports의 차이점
그럼 두 방식의 차이가 뭔지 알아보자. 일단 사용법이 조금 다르다.
* `exports`를 사용할 때는 `exports` 객체에 프로퍼티를 추가했다.
* `module.exports`를 사용할 때는 `module.exports` 변수에 아예 새로운 객체를 할당했다.

왜 두 방식의 사용법이 다른걸까? 이는 Node.js의 모듈 시스템에서 실제로 익스포트 되는 객체는 `module.exports`이고, `exports`는 이를 참조하는 변수에 불과하기 때문이다. 이를 확인하기 위해 다음 코드를 실행해보자.
```javascript
// exports_check.js
console.log(exports === module.exports);
console.log(module.exports);
```

```bash
$ node exports_check.js
true
{}
```

`module.exports`는 빈 오브젝트로 초기화되어 있으므로 `exports` 변수를 통해 프로퍼티를 추가할 수 있다. 하지만 `exports`에 새로운 객체를 할당하면 `module.exports`와는 상관 없는 변수가 되어버리므로 모듈이 동작하지 않는다.

```javascript
// square.js
exports = {
  area: (width) => width * width,
  perimeter: (width) => 4 * width
}
// 이제 exports는 module.exports가 아닌 다른 객체를 참조한다.
// module.exports는 여전히 빈 오브젝트이다.
```
square.js 파일을 위와 같이 수정하고 앱을 실행하면 아래와 같이 오류가 발생한다. 익스포트된 `square` 객체에 `area`라는 함수가 없기 때문이다.
```bash
$ node app.js
console.log(square.area(3)); // 9
                   ^
TypeError: square.area is not a function
```

Q. 그럼 반대로 아래와 같이 `module.exports`에 프로퍼티를 추가하는 것은 동작할까?
```javascript
// square.js
module.exports.area = (width) => width * width;
module.exports.perimeter = (width) => 4 * width;
```

### 요약
다음 사실을 기억하면 `exports`와 `module.exports`의 차이를 이해할 수 있다.
* Node.js에서 익스포트되는 객체는 `module.exports`이다.
* `module.exports` 빈 오브젝트(`{}`)로 초기화되어 있다.
* `exports`는 `module.exports`를 참조하는 변수이다.

## 참고 문서
[Mozilla - Express_Nodejs][1]
[Mozilla - Express_Nodejs Introduction][2]
[Node.js modules][3]


[1]: https://developer.mozilla.org/ko/docs/Learn/Server-side/Express_Nodejs
[2]: https://developer.mozilla.org/en-US/docs/Learn/Server-side/Express_Nodejs/Introduction#Importing_and_creating_modules
[3]: https://nodejs.org/api/modules.html#modules_modules
