---
title: Face Detection
date: 2022-03-14 21:23:22
toc: true
category: Computer Vision
tags:
    - Computer Vision
math: true
mermaid: true
author: wayexists02
---



Face detection이란, 이미지 또는 동영상에서 사람의 얼굴 부분을 찾아서 bounding box를 그려주는 작업을 말한다. 본 문서에서는 이러한 face detection이 어떻게 이루어지는지, face detection을 위해 사용되는 feature가 어떤 것이 있는지를 논의한다.



# Overview of Face Detection



Face detection은 이미지 또는 동영상으로부터 사람의 얼굴 부분을 찾아서 bounding box를 찾아주는 알고리즘들을 가리킨다. 이미지나 동영상을 촬영할 때, 보통 가장 중요하게 생각되는 객체는 사람으로, 특히, 사람의 얼굴이 이미지나 동영상의 중심이 되는 경우가 많다. 따라서, 이미지나 동영상을 촬영하거나, 이미지를 처리할 때, 사람의 얼굴 부분을 탐지하고 사람의 얼굴을 중심으로 이미지나 동영상을 처리하는 어플리케이션이 많다.

Face detection은 보통 다음과 같은 과정으로 이루어진다.

1. 이미지(또는 비디오 프레임) 한 장을 입력으로 받는다.
2. 슬라이딩 윈도우 기법으로 이미지 각 부분에 대해 feature를 추출한다.
3. 계산된 feature를 바탕으로 이미지의 각 부분이 사람의 얼굴인지, 얼굴이 아닌지 판별한다.
4. 사람의 얼굴이라고 판별된 부분의 bounding box를 리턴한다.

이때, 가장 중요한 부분이라고 할 수 있는 부분은 바로, 이미지의 각 부분에 대해 feature를 추출하는 것이라고 할 수 있다. 이때, feature는 사람의 얼굴 영역에서 나온 값과 사람의 얼굴이 아닌 영역에서 나온 값이 구분되는 feature여야 한다.

본 문서에서는 얼굴 탐지를 위해 가장 많이 사용되는 feature인 Haar feature를 살펴보고, Haar feature를 계산하기 위한 방법을 살펴본다.

여러 얼굴 이미지에 대해 Haar feature를 계산하고, 얼굴이 아닌 이미지에 대해 Haar feature가 계산되었다면, SVM과 같은 classifier를 학습할 수 있다. 학습된 SVM은 어떤 얼굴 이미지가 들어왔을 때, 얼굴이라고 판단할 수 있을 것이고, 얼굴이 아닌 이미지가 들어왔을 때에는 얼굴이 아니라고 판단할 수 있을 것이다.



# Haar Features for Face Detection



이미지에서 feature라고 하면, edge, boundary 또는 SIFT와 같은 feature가 가장 먼저 떠오르게 된다. 사람의 얼굴 부분은 edge 및 boundary가 복잡하지만, edge와 boundary가 복잡한 객체는 얼마든지 많으므로, 얼굴을 탐지하는 feature로 사용하기에는 부적합하다. SIFT의 경우, 두 영역간 similarity를 비교하는데 주로 뛰어난 성능을 보여줄 뿐, 어떤 영역을 특정 카테고리로 판별하는 데에는 적합하지 않다.

따라서, 이미지의 어떤 영역이 얼굴인지를 판별하기 위해서는 조금 다른 feature가 필요한데, 보통 얼굴을 탐지하는 데 있어서 가장 널리 사용되는 feature는 Haar feature이다.

Haar feature는 여러 스케일, 여러 가지의 Haar filter로 이미지의 각 영역에 적용한 후, 이들 필터들로 계산된 response의 합으로 계산된다.

<img src="https://user-images.githubusercontent.com/26874750/158171269-55a921a1-f592-40bc-8335-48864bd0c581.png" alt="Untitled" style="zoom:80%;" />

‘$*$’ 표시는 convolution을 의미하며, filter에서 흰 부분은 +1 값, 검은 부분은 -1 값을 의미한다.

하나의 Haar feature는 하나의 Haar filter를 이미지에 convolution하여 얻어진다. Haar filter는 large-scale image derivative라고 볼 수 있는데, 다음의 Haar filter를 예로 들어보자.

![Untitled 1](https://user-images.githubusercontent.com/26874750/158171329-5d75ee2c-c178-408c-aa32-9156a85a7c10.png)

이 Haar filter는 이미지에서 $x$축 방향의 edge에 민감하게 반응하는 edge detection filter(e.g. Sobel filter)와 유사한 모양이다. 즉, 이 Haar filter는 꽤 큰 크기의 $x$방향 edge detection filter라고 볼 수 있다.

마찬가지로, 다음의 Haar filter는 $y$축 방향으로의 큰 edge detection filter라고 볼 수 있다.

![Untitled 2](https://user-images.githubusercontent.com/26874750/158171361-27b5a8d5-bd47-465e-9a6d-4658467571e3.png)

다음의 Haar filter의 경우, 2차 미분을 이용한 edge detector인 Laplacian filter와 유사한 역할이라고 볼 수 있다.

![Untitled 3](https://user-images.githubusercontent.com/26874750/158171395-2bbcc884-d807-4821-9ee8-5ac638eb04a3.png)

Haar feature는 이처럼, 큰 스케일의 edge detection filter를 여러개 이용하여 추출한 feature를 의미한다. 이때, 다양한 스케일의 얼굴을 탐지하기 위해 위와 같은 Haar filter를 여러 스케일로 설계하게 된다.



## Computation Cost for Haar Features

Face detection은 카메라에 실시간으로 적용되는 경우가 많기에, 연산 비용이 적어야 한다. Haar feature를 계산하는 것은 convolution 곱을 직접 계산할 필요없이, (흰 영역의 픽셀 값들의 합) - (검은 영역의 픽셀 값들의 합)으로 간단하게 계산이 가능하기에 매우 효율적이다.

그럼에도 불구하고, Haar feature를 계산할 때, 모든 픽셀에 대해 계산해야 하며, Haar filter 역시 한두개가 아니다. 또한, Haar filter의 스케일도 다양하게 해야 하므로 그 연산 비용이 커지게 된다. Haar filter 하나에 대한 Haar feature 계산은 매우 효율적이지만, 전체적으로 보았을 때, 좀 더 속도를 개선할 필요가 있다. 이때, 사용될 수 있는 것이 integral image 방법이다.



### Integral Image

Integral image란, 모든 픽셀좌표 $(i, j)$에 대해 다음과 같이 정의된다.

$$
II(i, j) = \sum_{m=1}^i \sum_{n=1}^j I(m, n)
$$

이때, $I(m, n)$은 $(m, n)$ 좌표에서의 이미지 픽셀값이고, $II(i, j)$는 $(i, j)$ 좌표에서의 integral image 값이다.

그림으로 표현하면 다음과 같다.

![Untitled 4](https://user-images.githubusercontent.com/26874750/158171444-340902b4-94f9-402d-9c11-38fbb54265e5.png)

Integral image에서 $(i, j)$ 좌표에 들어갈 값은 원본 이미지에서 $(1, 1)$ 좌표를 좌상단 꼭지점으로 하고, $(i, j)$를 우하단 꼭지점으로 하는 윈도우 내 모든 픽셀값들의 합이다.

Haar feature를 계산하는 과정은 (흰 영역의 모든 픽셀값의 합) - (검은 영역의 모든 픽셀값의 합) 으로 계산되는데, Haar filter 윈도우 크기가 큰 경우, 더해야 하는 픽셀 개수가 많아지기 때문에 연산에 부담이 되었었다. 하지만, integral image가 있으면, 윈도우 크기와 상관없이 매우 적은 수의 덧셈으로 Haar feature를 계산할 수 있다.

예를들어, 어떤 이미지에서, 다음 윈도우 크기 내 모든 픽셀값들의 합을 계산하고 싶다고 하자.

![Untitled 5](https://user-images.githubusercontent.com/26874750/158171479-51544968-ac73-4a3e-a261-e6308d3b202d.png)

이 윈도우 내의 모든 픽셀값들의 합을 계산하는 과정은 픽셀 개수가 늘어남에 따라 연산량이 변한다. 또한, Haar filter 윈도우 크기는 기본적으로 크기 때문에, 연산해야 하는 덧셈 수는 매우 많다.

하지만, integral image가 있는 경우, 이야기가 달라진다. Integral image가 있는 경우, 위 그림에서의 윈도우 내의 모든 픽셀값을 계산하는 과정은 단 3번의 덧셈으로 가능하다. 윈도우 내의 모든 픽셀값의 합은 다음 integral image에서, $(a - b - c + d)$와 같다.

![Untitled 6](https://user-images.githubusercontent.com/26874750/158171517-a4c73f74-8121-4373-941b-5e74b60b9800.png)

윈도우 내의 모든 픽셀값들의 합은 이처럼 단 세번의 덧셈으로 가능하며, 윈도우 크기가 크더라도 덧셈은 3번이면 충분하다. 다음 Haar filter를 이용하여 Haar feature를 계산하는 과정은 일곱번의 덧셈이면 충분하다.

![Untitled 7](https://user-images.githubusercontent.com/26874750/158171552-55b3a77d-4efe-4bf3-98b9-85a9a6c0dd86.png)

위 그림에서 왼쪽 그림의 Haar filter를 이용하여 Haar feature를 계산하는 과정은, integral image에서 다음을 계산하는 것과 같다.

$$
\text{Haar Feature} = (b - c - e + f) - (a - b - d + e)
$$

Integral image가 있으면 Haar feature를 매우 적은 수의 덧셈으로 계산이 가능하다는 것을 알았는데, Integral image를 계산하는 과정이 복잡하면 의미가 없다. 하지만, integral image 역시 매우 빠르게 효율적으로 계산이 가능하다.



### Integral Image Computation

Integral image를 계산하는 과정은 다음과 같다.

1. Integral image $II$의 모든 픽셀값을 0으로 초기화한다.

2. Integral image의 $(1, 1)$ 좌표부터 $(h, w)$ 좌표(이미지 우하단 좌표)까지 왼쪽에서 오른쪽으로, 위에서 아래로 훑으면서 다음을 실행한다.

   $$
   II(i, j) = I(i, j) + II(i - 1, j) + II(i, j - 1) - II(i - 1, j - 1)
   $$

그림으로 표현하면 다음과 같다.

![Untitled 8](https://user-images.githubusercontent.com/26874750/158171595-fa0b810f-54e4-4260-b295-8e2ee30aebf1.png)

연한 노랑색 부분은 이미 계산이 완료된 부분이며, 현재 $a$ 픽셀값에서의 integral image 값을 계산하고 있다고 하자. 이때, $b, c, d$는 이미 integral image 값이 계산이 완료되었다.

$a$ 위치에서의 integral image 값은 다음과 같이 계산된다.

$$
II(a) = I(a) + II(b) + II(c) - II(d)
$$

즉, 이미지의 모든 픽셀을 한번씩 쭉 훑으면 integral image가 계산된다. Integral image는 한 번 계산해놓으면 모든 Haar feature 계산에 이용할 수 있어서 매우 효율적이다.