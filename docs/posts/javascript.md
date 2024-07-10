---
date: 2024-07-10
categories:
  - Computer Science
---

# Javascript

## This

자바스크립트 입문자가 가장 어려워하는 내용으로 `this`를 빼놓을 수 없다.
함수를 호출하는 시점에 따라 this 키워드가 가리키는 값이 다르기 때문이다.

<!-- more -->

```js
const computer = {
    time: 5,
    reboot: function () {  // arrow function
        let t = this.time;
        console.log(`Rebooting in ${t} seconds...`);
    }
}

computer.reboot();  // Rebooting in 5 seconds...
```

위 코드에서 this는 익명 함수 안쪽에 있지만, 호출 시점은 코드 맨 마지막 줄 `computer.reboot()` 이다.
호출 시점에 함수는 `computer` 객체에 포함되므로, `this.time` 표현식은 컴퓨터 객체의 `time: 5` 속성을 가리킨다.
하지만 화살표 함수를 사용하게 되면 어떨까?

```js
const computer = {
    time: 5,
    reboot: () => {  // arrow function
        let t = this.time;
        console.log(`Rebooting in ${t} seconds...`);
    }
}

computer.reboot();  // Rebooting in undefined seconds...
```

이전 코드에서 익명 함수를 화살표 함수로 바꾸기만 했는데 실행 결과가 달라졌다!
화살표 함수 안에서 this 키워드가 무엇을 가리키는지 알아보자.

```js
const computer = {
    reboot: () => {
        console.log(this);  // Window {window: Window, self: Window, ...}
    }
}
computer.reboot();
```

놀랍게도 전역 객체를 가리킨다. (window를 가리키지만 node.js 환경이라면 global)

MDN 레퍼런스를 인용하면 화살표 함수에서 this 키워드는 부모, 혹은 바깥쪽 scope를 계승한다고 한다.
직관적으로 생각하면 함수 바깥쪽은 computer니까 컴퓨터 객체를 계승하지 않을까?

아니다! 객체 리터럴 `obj = {...};` 표현식은 scope를 형성하지 않는다. 따라서 전역 스코프를 계승하게 되어 this가 window를 가리킨다.

```js
class Computer {
    time = 5;
    reboot = () => {
        let t = this.time;
        console.log(`Rebooting in ${t} seconds...`);
    }
}

new Computer().reboot();  // Rebooting in 5 seconds...
```

여담으로 화살표 함수를 메소드 선언에 사용하면 this 키워드는 `Computer` 객체를 잘 가리킨다.
클래스는 객체마다 고유 scope를 형성하기 때문에 this는 전역 객체를 가리키지 않는다. 

정리하면 this 키워드는 호출 시점에 따라 가리키는 객체가 유동적으로 변하지만,
화살표 함수에서 this 키워드는 바깥쪽 scope에 해당하는 맥락을 유지한다고 볼 수 있다.
