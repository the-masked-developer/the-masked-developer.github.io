---

title      : 자료구조와 객체 (1) - 추상화와 다형성
date       : 2021-11-13 14:45:26 +0900
tags       : 
categories : 프로그래밍
draft      : true
author     : 이민혁

---

## 추상화의 수준의 차이

자료구조는 데이터를 노출하고 객체는 추상화한다. 이 차이는 아래의 다형성의 차이를 만들어내게 된다.


## 다형성의 차이

아래와 같은 두 클래스 `Rectangle` 와 `Circle` 이 있을 때,

```java
public class Rectangle {
  public float x1;
  public float x2;
  public float y1;
  public float y2;
}

public class Circle {
  public float x;
  public float y;
  public float radius;
}
```

만약 두 클래스의 면적을 구하고 싶다면? 만약 두 클래스를 자료구조로 취급한다면 (struct?) 아래와 같은 helper class 를 만들게 된다.

```java
public class GeometryHelper {

  public double calcArea(Object obj) {
    if (obj instanceof Rectangle) {
      // ...
    } else if (obj instanceof Circle) {
      // ...
    }
    throw new Exception("No match");
  }
}
```

코드는 객체지향이라기보다는 절차지향적이다. 만약 위 코드를 객체지향적으로 만든다면, `Shape` 와 같은 인터페이스를 정의하거나 상속받은 후 `getArea()` 와 같은 method 를 구현한다. 

```java
public class Rectangle implements Shape {
  public float x1;
  public float x2;
  public float y1;
  public float y2;
  
  public float getArea() {
    // ...
  }
}

public class Circle implements Shape {
  public float x;
  public float y;
  public float radius;
  
  public float getArea() {
    // ...
  }
}
```

위 코드는 객체지향적이 되었다. 

이제 새로운 함수(둘레 또는 지름) 를 추가하거나 새로운 도형(타입)이 생길 수도 있다. 새로운 함수를 만들 때, 객체지향적인 코드는 모든 객체에 그 함수를 구현해야 한다. 즉 모든 객체에 대한 수정이 불가피하다. 새로운 도형을 만들 때는 기존 도형은 내버려 두어도 괜찮다. 절차지향적인 코드에서는 기존 체는 내버려둔채 새로운 함수를 구현하면 된다. 반면 새로운 도형을 만들 때는 모든 함수들에서 고려해야 한다. 
* 객체지향적인 코드에서는 모든 도형에 새로운 함수를 추가하거나 새로운 도형을 만들면 된다. 즉, 기존 함수들은 수정할 일이 없다.
* 절자지향적인 코드에서는 GeometryHelper 에 함수만 수정하거나 추가하면 된다. 즉 기존 도형(자료구조) 를 변경하지 않는다.


## 결론

> 절차적인 코드는 새로운 자료구조를 추가하기 어렵다. (모든 함수를 고쳐야 한다.) 반면 객체지향적인 코드는 새로운 함수를 추가하기 어렵다. (모든 클래스를 고쳐야한다.)

**위 두가지는 정확히 Trade off 관계로, 때로는 객체보다 단순한 자료구조와 절차지향적인 코드가 더 효과적일 수도 있다.** 그러므로, 앞으로 새로운 타입이 필요해질 것 같은 지, 함수가 필요하게 될 지를 고려해서 절하게 사용하면 된다. 

