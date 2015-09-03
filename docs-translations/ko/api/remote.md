﻿# remote

`remote` 모듈은 메인 프로세스와 랜더러 프로세스(웹 페이지) 사이의 inter-process (IPC) 통신을 간단하게 추상화 한 모듈입니다.

Electron의 랜더러 프로세스에선 GUI와 관련 없는 모듈만 사용할 수 있습니다.
기본적으로 랜더러 프로세스에서 메인 프로세스의 API를 사용하려면 메인 프로세스와 inter-process 통신을 해야 합니다.
하지만 `remote` 모듈을 사용하면 따로 inter-process 통신을 하지 않고 직접 명시적으로 모듈을 사용할 수 있습니다.
Java의 [RMI](http://en.wikipedia.org/wiki/Java_remote_method_invocation)와 개념이 비슷합니다.

다음 예제는 랜더러 프로세스에서 브라우저 창을 만드는 예제입니다:

```javascript
var remote = require('remote');
var BrowserWindow = remote.require('browser-window');

var win = new BrowserWindow({ width: 800, height: 600 });
win.loadUrl('https://github.com');
```

**참고:** 반대로 메인 프로세스에서 랜더러 프로세스에 접근 하려면 [webContents.executeJavascript](browser-window.md#webcontents-executejavascript-code) 메서드를 사용하면 됩니다.

## Remote 객체

`remote` 모듈로부터 반환된 각 객체(메서드 포함)는 메인 프로세스의 객체를 추상화 한 객체입니다. (우리는 그것을 remote 객체 또는 remote 함수라고 부릅니다)
Remote 모듈의 메서드를 호출하거나, 객체에 접근하거나, 생성자로 객체를 생성하는 등의 작업은 실질적으로 동기형 inter-process 메시지를 보냅니다.

위의 예제에서 사용한 두 `BrowserWindow`와 `win`은 remote 객체입니다. 그리고 `new BrowserWindow`이 생성하는 `BrowserWindow` 객체는 랜더러 프로세스에서 생성되지 않습니다.
대신에 이 `BrowserWindow` 객체는 메인 프로세스에서 생성되며 랜더러 프로세스에 `win` 객체와 같이 이에 대응하는 remote 객체를 반환합니다.

## Remote 객체의 생명 주기

Electron은 랜더러 프로세스의 remote 객체가 살아있는 한(다시 말해서 GC(garbage collection)가 일어나지 않습니다) 대응하는 메인 프로세스의 객체는 릴리즈되지 않습니다.
Remote 객체가 GC 되려면 대응하는 메인 프로세스 내부 객체의 참조가 해제되어야만 합니다.

만약 remote 객체가 랜더러 프로세스에서 누수가 생겼다면 (예시: 맵에 저장하고 할당 해제하지 않음) 대응하는 메인 프로세스의 객체도 누수가 생깁니다.
그래서 remote 객체를 사용할 땐 메모리 누수가 생기지 않도록 매우 주의해서 사용해야 합니다.

참고로 문자열, 숫자와 같은 원시 값 타입은 복사에 의한 참조로 전달됩니다.

## 메인 프로세스로 콜백 넘기기

메인 프로세스의 코드는 `remote` 모듈을 통해 랜더러 프로세스가 전달하는 콜백 함수를 받을 수 있습니다.
하지만 이 작업은 반드시 주의를 기울여 사용해야 합니다.

첫째, 데드락을 피하기 위해 메인 프로세스로 전달된 콜백들은 비동기로 호출됩니다.
이러한 이유로 메인 프로세스로 전달된 콜백들의 반환 값을 내부 함수에서 언제나 정상적으로 받을 것이라고 예측해선 안됩니다.

예를 들어 메인 프로세스에서 `Array.map` 같은 메서드를 사용할 때 랜더러 프로세스에서 전달된 함수를 사용해선 안됩니다:

```javascript
// mapNumbers.js 메인 프로세스
exports.withRendererCallback = function(mapper) {
  return [1,2,3].map(mapper);
}

exports.withLocalCallback = function() {
  return exports.mapNumbers(function(x) {
    return x + 1;
  });
}
```

```javascript
// 랜더러 프로세스
var mapNumbers = require("remote").require("mapNumbers");

var withRendererCb = mapNumbers.withRendererCallback(function(x) {
  return x + 1;
})

var withLocalCb = mapNumbers.withLocalCallback()

console.log(withRendererCb, withLocalCb) // [true, true, true], [2, 3, 4]
```

보다시피 랜더러 콜백의 동기 반환 값은 예상되지 않은 처리입니다.
그리고 메인 프로세스에서 처리한 함수의 반환 값과 일치하지 않습니다.

둘째, 콜백들은 메인 프로세스로 전달, 호출된 이후에도 자동으로 함수의 참조가 릴리즈 되지 않습니다.
함수 참조는 메인 프로세스에서 GC가 일어나기 전까지 계속 프로세스에 남아있게 됩니다.

다음 코드를 보면 느낌이 올 것입니다. 이 예제는 remote 객체에 `close` 이벤트 콜백을 설치합니다:

```javascript
var remote = require('remote');

remote.getCurrentWindow().on('close', function() {
  // blabla...
});
```

하지만 이 코드 처럼 이벤트를 명시적으로 제거하지 않는 이상 콜백 함수의 참조가 계속해서 메인 프로세스에 남아있게 됩니다.
만약 명시적으로 콜백을 제거하지 않으면 매 번 창을 새로고침 할 때마다 콜백을 새로 설치합니다.
게다가 이전 콜백이 제거되지 않고 계속해서 쌓이면서 메모리 누수가 발생합니다.

설상가상으로 이전에 설치된 콜백의 콘텍스트가 릴리즈 되고 난 후(예: 페이지 새로고침) `close` 이벤트가 발생하면 예외가 발생하고 메인 프로세스가 작동 중지됩니다.

이러한 문제를 피하려면 랜더러 프로세스에서 메인 프로세스로 넘긴 함수의 참조를 사용 후 확실하게 제거해야 합니다.
작업 후 이벤트 콜백을 포함하여 책임 있게 함수의 참조를 제거하거나 메인 프로세스에서 랜더러 프로세스가 종료될 때 내부적으로 함수 참조를 제거하도록 설계해야 합니다.

## Methods

`remote` 모듈은 다음과 같은 메서드를 가지고 있습니다:

### `remote.require(module)`

* `module` String

메인 프로세스의 `require(module)` API를 실행한 후 결과 객체를 반환합니다.

### `remote.getCurrentWindow()`

현재 웹 페이지가 들어있는 [`BrowserWindow`](browser-window.md) 객체를 반환합니다.

### `remote.getCurrentWebContents()`

현재 웹 페이지의 [`WebContents`](web-contents.md) 객체를 반환합니다.

### `remote.getGlobal(name)`

* `name` String

메인 프로세스의 전역 변수(`name`)를 가져옵니다. (예시: `global[name]`)

### `remote.process`

메인 프로세스의 `process` 객체를 반환합니다. `remote.getGlobal('process')`와 같습니다. 하지만 캐시 됩니다.