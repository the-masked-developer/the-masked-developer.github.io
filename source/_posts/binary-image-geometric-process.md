---
title: Binary Image Processing (Geometric Information)
date: 2022-01-23 12:23:22
index_img:
category: Computer Vision
tags:
    - Computer Vision
math: true
mermaid: true
author: wayexists02
---

# 

# Geometric Information

바이너리 이미지에서 추출할 수 있는 첫 번째 주요 정보는 객체에 대한 geometric 정보이다. 본 문서는 바이너리 이미지에서 추출할 수 있는 geometric 정보에 대해 다룬다.

# Geometric Information of Binary Images

바이너리 이미지로 얻을 수 있는 이미지의 특징 중 하나는 이미지 내의 객체에 대한 geomatric 정보이다. 여기서, 객체의 geometric 정보란 다음과 같은 정보들을 말한다.

- 이미지 내에서 객체의 넓이(area)
- 이미지 내에서 객체의 위치 좌표(coordinates)
- 이미지 내에서 객체의 방향(direction)

이미지를 binarization할 수 있다면, 위와 같은 이미지의 geometric 특징을 추출할 수 있다.

## Area of Objects

Gray-scale 이미지 $g$에 대해 추출한 바이너리 이미지를 $b$라고 할때, 이미지 속 객체의 넓이(area) $A$는 다음과 같이 계산할 수 있다.

$$
A = \int \int b(x, y) dx dy
$$

이때, $b(x, y)$는 $(x, y)$좌표에서의 픽셀의 바이너리 값으로 객체에 해당하는 픽셀만 1, 나머지 영역은 0이 된다.

하지만, 이미지 픽셀 사이의 공간은 discrete 이므로, 다음과 같이 객체의 넓이를 계산해야 한다.

$$
A = \sum_{x} \sum_{y} b(x, y)
$$

객체의 넓이는 이미지 내에서 객체가 움직이던, 회전하던 변하지 않는 특징이 될 수 있다(3차원적으로 회전하는 것 제외). 그러나, 이미지 내에 객체가 여러개인 경우, classification(or segmentation)을 먼저 수행할 필요가 있다.

참고로, 바이너리 이미지에서 객체의 넓이는 zero moment에 해당한다.

## Position of Objects

바이너리 이미지 $b$에서, 객체에 해당하는 픽셀이 모두 1이고, 배경에 해당하는 픽셀이 모두 0일때, 객체의 중심 좌표는 다음과 같이 계산한다.

$$
\bar{x} = \frac{1}{A} \int \int x b(x, y) dx dy \\
\bar{y} = \frac{1}{A} \int \int y b(x, y) dx dy
$$

이때, $A$는 객체의 넓이로, 바이너리 이미지에서 객체에 해당하는 픽셀의 개수이다.

역시, 픽셀공간이 discrete이기 때문에 다음과 같이 객체의 위치를 계산해야 한다.

$$
\bar{x} = \frac{1}{A} \sum_x \sum_y x b(x, y) \\
\bar{y} = \frac{1}{A} \sum_x \sum_y y b(x, y)
$$

객체의 위치는 first moment에 해당한다.

## Direction of Objects

객체의 방향이란, 객체의 모양에서 가장 긴 축을 말한다. 예를들어, 다음 항아리 모양의 객체에서 객체의 방향은 그림의 축과 같다.

![Untitled](https://user-images.githubusercontent.com/26874750/150641155-1fb06222-e384-49c1-be1b-e8eddbe80203.png)

바이너리 이미지에서 객체의 방향을 구하는 아이디어는 “2nd central moment가 가장 작은 축”을 구하는 것이다.

2nd central moment란, 어떤 축이 있을 때, 해당 축과 각 픽셀과의 거리의 제곱합을 의미하며, $(x, y)$좌표에 있는 점과 축과의 거리를 $r_{x, y}$라고 했을 때, 다음과 같이 정의된다.

$$
E = \int \int r_{x,y}^2 b(x, y) dx dy
$$

다음과 같은 두 개의 축을 생각해보자.

![Untitled 1](https://user-images.githubusercontent.com/26874750/150641166-92b6aff4-d532-45c4-8324-52c890246c21.png)

왼쪽 그림에서는 객체의 모든 픽셀이 축과의 거리가 다 가까우므로 $E$ 값이 작다. 반면, 오른쪽 그림에서는 객체의 일부 픽셀이 축과의 거리가 상당히 먼 경우가 많기 때문에, $E$값이 커지게 된다. 즉, $E$값이 최소가 되는 축을 찾으면 객체의 방향 축을 찾을 수 있게 된다(PCA가 생각나는데?).

우리가 어떤 직선을 정의할때 보통 다음과 같이 직선의 방정식을 이용하게 된다.

$$
y = mx + b
$$

그러나, 이런 형태의 식에서는 $m$의 범위가 $-\infty$에서 $\infty$가 되기 때문에, 최적화가 좀 어렵다. 따라서, 위와 같은 직선의 방정식 말고 직선의 각도와 원점과의 거리를 이용하여 직선을 reparameterization하여 다음과 같이 정의한다.

$$
x \sin \theta - y \cos \theta + \rho = 0
$$

이때, $\theta$는 해당 직선과 $x$축 사이의 각도이며, $\rho$는 직선과 원점(0, 0) 사이의 거리이다.

이 직선과 $(x, y)$ 좌표의 픽셀 까지의 거리 $r_{x, y}$는 다음과 같다.

$$
r_{x, y} = \frac{|x \sin \theta - y \cos \theta + \rho|}{\sqrt{\sin^2 \theta + \cos^2 \theta}} = |x \sin \theta - y \cos \theta + \rho|
$$

이에 따라, 2nd central moment $E$는 다음과 같이 변형될 수 있다.

$$
E = \int \int |x \sin \theta - y \cos \theta + \rho|^2 b(x, y) dx dy
$$

우리가 원하는 것은 이 $E$를 최소화하는 축을 찾는 것이므로, $E$를 최소화하는 $\theta$와 $\rho$를 찾는 것이 우리가 계산해야 할 것이다.

위 $E$를 $\rho$에 대해 미분하여 0이 되는 “극점”을 찾아보자.

$$
\frac{d}{d\rho} E = \int \int 2 |x \sin \theta - y \cos \theta + \rho| b(x, y) dx dy = 0 \\
= 2 (A \bar{x} \sin \theta - A \bar{y} \cos \theta + A \rho) = 0 \\
2A (\bar{x} \sin \theta - \bar{y} \cos \theta + \rho) = 0
$$

이때, $A$는 객체의 넓이이고, $\bar{x}, \bar{y}$는 객체의 중심점 좌표이다. 객체의 넓이는 0이 아니므로, 직선 $x \sin \theta - y \cos \theta + \rho = 0$ 이 객체의 방향 축이 되려면 다음을 만족해야 한다.

$$
\bar{x} \sin \theta - \bar{y} \cos \theta + \rho = 0
$$

즉, 직선 $x \sin \theta - y \cos \theta + \rho = 0$는 객체의 중심 $(\bar{x}, \bar{y})$를 지나는 직선이어야 한다. 이것을 반영하여 직선을 다음과 같이 수정하자.

$$
(x - \bar{x}) \sin \theta - (y - \bar{y}) \cos \theta = 0
$$

이때, $E$를 다시 적어보면 다음과 같다.

$$
E = \int \int \vert x \sin \theta - y \cos \theta + \rho \vert^2 b(x, y) dx dy
$$

$$
= \int \int \vert(x - \bar{x}) \sin \theta - (y - \bar{y}) \cos \theta \vert^2 b(x, y) dx dy
$$

$$
= \int \int ((x - \bar{x})^2 \sin^2 \theta + (y - \bar{y})^2 \cos^2 \theta - 2(x - \bar{x})(y - \bar{y})\sin \theta \cos \theta) b(x, y) dx dy
$$

$$
= \sin^2 \theta \int (x - \bar{x})^2 dx+ \cos^2 \theta \int (y - \bar{y})^2 dy- 2 \sin \theta \cos \theta \int \int (x - \bar{x})(y - \bar{y}) dx dy
$$

이때, $\bar{x}, \bar{y}$는 쉽게 구할 수 있으므로, 맨 마지막 식의 integral은 간단하게(?) 계산이 가능하다. $E$식을 좀 간단하게 적기 위해 다음을 정의하자.

$$
a := \int (x - \bar{x})^2 dx \\
b := \int (y - \bar{y})^2 dy \\
c := \int \int (x - \bar{x})(y - \bar{y}) dx dy
$$

이때, $E$는 다음과 같이 변형할 수 있다.

$$
E = a \sin^2 \theta + b \cos^2 \theta - 2c \sin \theta \cos \theta
$$

이제, 이 식을 다시 $\theta$에 대해 미분하여 0이 되는 지점을 찾아 $E$의 “극점”을 구해보자.

$$
\frac{d}{d\theta} E = 2a\sin\theta \cos \theta - 2b \sin \theta \cos \theta - 2c(\cos^2 \theta - \sin^2 \theta) = 0 \\
= (a - b) \sin 2\theta - 2c \cos 2\theta = 0 \\
\tan 2 \theta = \frac{2c}{a - b}
$$

즉, 마지막 식이 만족할 때, $E$가 최소 또는 최대가 된다. 그런데, $0 \leq \theta \leq 2\pi$ 범위에서, 방정식의 해는 2개가 되는데, 어떤 $\theta_1$에 대해, $\tan 2 \theta_1 = \tan (2 \theta_1 + \pi)$ 이기 때문이다. 이때, 하나는 최소에 대한 해, 하나는 최대에 대한 해일 것인데, 2차 미분을 통해 어느 점이 최소점인지 찾아야 한다. $E$의 2차 미분은 다음과 같다.

$$
\frac{d^2}{d\theta^2} E = 2(a - b) \cos 2\theta + 4c \sin 2\theta
$$

이 식에 $\theta_1$와 $\theta_1 + \frac{1}{2}\pi$를 넣어보고, 2차미분값이 양의 값인 부분인 $\theta$를 찾으면 된다. 여기까지, 직선의 방정식의 파라미터 $\rho, \theta$를 구한 것이며, 객체의 방향 축을 구하는 과정이었다.

결과적으로, $\theta$는 다음과 같다.

$$
\theta = \frac{1}{2}\arctan\frac{2c}{a - b}
$$

## 

## Roundness

객체가 얼마나 둥글둥글한지를 측정하는 지표로, 2nd central moment $E$의 최댓값 $E_{\max}$과 최솟값 $E_{\min}$으로 구할 수 있다.

$$
\text{roundness} = \frac{E_{\min}}{E_{\max}}
$$

이 값이 1에 가까우면 객체가 둥글둥글 한 것이고, 0에 가까우면 객체가 길쭉한 타원에 가까운 것이다.