---
title: Intro to Binary Images
date: 2022-01-23 12:22:22
toc: true
category: Computer Vision
tags:
    - Computer Vision
math: true
mermaid: true
author: wayexists02
---

# 

# Introduction to Binary Images

바이너리 이미지란, 한개 또는 두 개의 숫자로만 이미지 값으로 존재하는 이미지를 말한다. 흔히, 완전 흰색과 완전 흑색으로만 구성되기도 한다. 바이너리 이미지는 다루기 쉬우면서도 이미지로부터 추출할 수 있는 매우 강건한 특징(feature)중 하나이다. 바이너리 이미지를 일단 얻게 되면, 이미지를 segmentation 하거나 분류하는 등의 작업을 수행할 수 있다.

# Binary Images

바이너리 이미지는 하나 또는 두 개의 값(보통 0과 1)으로 구성된 이미지이다.

바이너리 이미지 $b$는 gray-scale 이미지 $g$를 thresholding을 수행함으로서 얻을 수 있다.

- $g(x, y) < T$ 이면, $b(x, y) = 0$
- $g(x, y) \geq T$ 이면, $b(x, y) = 1$

바이너리 이미지는 이미지 속 객체의 structure를 파악하는데 매우 중요한 역할을 할 수 있으며, 객체 segmentation을 매우 뛰어나게 수행할 수 있다는 장점이 있다.

하지만, threshold $T$를 주의깊게 설정해야 하는데, 너무 낮게 설정하면 모두 흰색인 바이너리 이미지가 얻어질 것이고, 너무 높게 설정하면 모두 검은색인 바이너리 이미지가 얻어질 것이다. Threshold를 정하는 방법은 임의로 정하는 방법도 있지만, color histogram을 이용하는 방법도 있다.

Color histogram을 이용하여 threshold를 정하는 방법은 이미지에서 foreground와 background가 명확하게 구분되는 이미지에서만 가능한데, 이러한 이미지에서 color histogram을 그린 후, color histogram의 골짜기 부분을 threshold로 삼는 것이다.

![Untitled](https://user-images.githubusercontent.com/26874750/150640898-2e231d83-a220-4186-aab5-ff48e62ea3d4.png)

골짜기 부분을 threshold로 삼게 되면 foreground 객체는 하얗게, background 객체는 검게 binarize될 것이다.

이미지로부터 바이너리 이미지를 얻어내는 작업은 때로 어려울 수 있다. 배경이 다양하거나 객체의 색상 또는 명암이 배경과 비슷한 경우, threshold를 하나로 정하기 힘든 경우가 많다. 이 경우, 바이너리 이미지를 얻는 것 대신 다른 방법으로 이미지의 특징을 추출할 필요가 있다. 이처럼 바이너리 이미지를 계산할 수 있는 경우는 한정적이지만, 바이너리 이미지를 계산할 수 있다면, 매우 강건한 이미지의 특징을 얻을 수 있다.

바이너리 이미지로부터 추출할 수 있는 중요한 특징은 다음과 같이 크게 세 가지가 존재한다.

1. 객체에 대한 geometric 정보
2. 객체에 대한 세그멘테이션 마스크
3. 객체에 대한 구조적 정보(스켈레톤 등)

이후 문서에서 위 세 가지 정보를 추출하는 방법에 대해 다루고자 한다.

- [Binary Image Processing (Geometric Information)](https://the-masked-developer.github.io/wiki/binary-image-geometric-process/)