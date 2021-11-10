---
title: 타입스크립트로 좋은 테스트 작성하기
date: 2021-11-09 22:22:22
index_img:
category: 글쓰기
tags:
  - 글쓰기
math: true
mermaid: true
sticky: 100
author: maskman
---
## 좋은 테스트 코드를 작성하는 원칙

깨끗한 테스트 코드는 신뢰할 수 있는 개발과 배포를 가능하게 합니다!


### 직관적이고 이해가 되는 테스트 이름으로 구성하기
---

- 테스트 이름을 추상적으로 쓰지마세요.
- 정확히 무엇을 테스트 하는지 명시적으로 작성하세요.

🟥 **안좋은 예**

```jsx
describe("주문 검증", () => {
  const postOrder = { ... };
  const order = await service.addOrder(postOrder);
  expect(order).toEqual()
});
```

---

**🟦 좋은 예**

```jsx
describe("정상적인 주문이 추가되는지 검증합니다.", () => {
  const postOrder = { ... };
  const order = await service.addOrder(postOrder);
  expect(order).toEqual()
});
```

### 글로벌 테스트 픽스처를 작성하지 마세요

- 복잡하고 이해하기 힘든 테스트 픽스처는 쉽게 추론하기가 힘듭니다.
- 테스트의 결합도가 증가하는 건 좋지 않습니다.
- 반복 사용되는 픽스처가 있다면, 외부에 static 팩토리 메소드를 만드세요.

🟥 **안좋은 예**

```jsx
beforeAll(async () => {
	const user = await service.addUser();
	const money = await service.addMoney(user);
	await service.addOrder(user, money);
});
describe("주문을 취소합니다", () => {
  const order = await service.deleteOrder();
  expect(order.status).toBe("canceled")
});
```

---

**🟦 좋은 예**

```jsx
class TestFactory {  
	static create() {
		return new ~~()
	}
}

describe("정상적인 주문이 추가되는지 검증합니다.", () => {
	const user = TestFactory.createUser()
	const money = TestFactory.createMoney() 
  const order = await service.addOrder(user, money);
  expect(order.status).toBe("canceled")
});
```

### 테스트를 반드시 격리시키세요

- 하나의 테스트는 반드시 격리되어 실행되어 외부에 영향을 받지 않아야 합니다.

🟥 **안좋은 예**

```jsx
let addOrder
describe("주문을 추가합니다", () => {
  const order = await service.addOrder();
	addOrder = order;
  expect(order.status).toBe("added")
});

describe("주문을 취소합니다", () => {
  const order = await service.deleteOrder(addOrder);
  expect(order.status).toBe("canceled")
});
```

---

**🟦 좋은 예**

```jsx
describe("주문을 추가합니다", () => {
  const order = await service.addOrder();
  expect(order.status).toBe("added")
});

describe("주문을 취소합니다", () => {
  const addOrder = await service.addOrder();
  const order = await service.deleteOrder(addOrder);
  expect(order.status).toBe("canceled")
});
```