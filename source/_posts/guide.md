---
title: 글 작성 가이드
date: 2021-11-09 22:22:22
index_img: https://t1.daumcdn.net/cfile/tistory/993FB53F5C85F30218
category: 글쓰기
tags:
  - 글쓰기
math: true
mermaid: true
sticky: 100
author: maskman
---

## 글 작성 가이드

사용할 수 있는 마크다운 문법 및 추가적인 플러그인 사용법에 관해서 기술

<!-- more -->

emphasis: `keyword`

highlight.js

```python
def fib(n):
    a, b = 0, 1
    while a < n:
        print(a, end=' ')
        a, b = b, a+b
    print()
fib(1000)
```

```go
type Map struct {
    mu Mutex
    read atomic.Value
    dirty map[interface{}]*entry
    misses int
}
```

## Table

| Header 1 | Header 2 | Header 3  |
| -------- | -------- | --------- |
| Key 1    | Value 1  | Comment 1 |
| Key 2    | Value 2  | Comment 2 |
| Key 3    | Value 3  | Comment 3 |

## H2

### H3

번호가 있는 목록

1. A
2. B
3. C

### H3

번호가 없는 목록

- A
- B
- C

## 이미지

![](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e1/Musee_de_la_bible_et_Terre_Sainte_001.JPG/440px-Musee_de_la_bible_et_Terre_Sainte_001.JPG)

## LaTex

$$
\Gamma _ { \epsilon } ( x ) = [ 1- e ^ { - 2\pi \epsilon } ] ^ { 1- x } \prod _ { n = 0} ^ { \infty } \frac { 1- \operatorname{exp} ( - 2\pi \epsilon ( n + 1) ) } { 1- \operatorname{exp} ( - 2\pi \epsilon ( x + n ) ) }
$$

$$
\left( \begin{array} c t ^ { \prime } \\ x ^ { \prime } \\ y ^ { \prime } \\ z ^ { \prime } \end{array} \right) = \left( \begin{array} { c c c c } { \gamma } & { - \gamma \beta } & { 0 } & { 0 } \\ { - \gamma \beta } & { \gamma } & { 0 } & { 0 } \\ { 0 } & { 0 } & { 1 } & { 0 } \\ { 0 } & { 0 } & { 0 } & { 1 } \end{array} \right) \left( \begin{array} c t \\ x \\ y \\ z \end{array} \right)
$$

$$
6 \mathrm { CO } _ { 2 } + 6 \mathrm { H } _ { 2 } \mathrm { O } \rightarrow \mathrm { C } _ { 6 } \mathrm { H } _ { 12 } \mathrm { O } _ { 6 } + 6 \mathrm { O } _ { 2 }
$$

## mermaid flowchart

```mermaid
sequenceDiagram
participant Alice
participant Bob
Alice->>John: Hello John, how are you?
loop Healthcheck
    John->>John: Fight against hypochondria
end
Note right of John: Rational thoughts <br/>prevail...
John-->>Alice: Great!
John->>Bob: How about you?
Bob-->>John: Jolly good!
```

```mermaid
gantt
dateFormat  YYYY-MM-DD
title Adding GANTT diagram to mermaid

section A section
Completed task            :done,    des1, 2014-01-06,2014-01-08
Active task               :active,  des2, 2014-01-09, 3d
Future task               :         des3, after des2, 5d
Future task2               :         des4, after des3, 5d
```

```mermaid
classDiagram
Class01 <|-- AveryLongClass : Cool
Class03 *-- Class04
Class05 o-- Class06
Class07 .. Class08
Class09 --> C2 : Where am i?
Class09 --* C3
Class09 --|> Class07
Class07 : equals()
Class07 : Object[] elementData
Class01 : size()
Class01 : int chimp
Class01 : int gorilla
Class08 <--> C2: Cool label
```

### note

{% note info %}
Info `markdown`
{% endnote %}

{% note warning %}
Warning `markdown`
{% endnote %}

{% note primary %}
etc `markdown`
{% endnote %}

### label

{% label info @info %} {% label warning @warning %} {% label primary @primary %}

### checkbox

{% cb checkbox, true %}
{% cb checkbox, false %}

### js

{% btn javascript:window.alert('hi');, button %}

### 이미지 분할

{% gi 5 3-2 %}
![](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e1/Musee_de_la_bible_et_Terre_Sainte_001.JPG/440px-Musee_de_la_bible_et_Terre_Sainte_001.JPG)
![](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e1/Musee_de_la_bible_et_Terre_Sainte_001.JPG/440px-Musee_de_la_bible_et_Terre_Sainte_001.JPG)
![](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e1/Musee_de_la_bible_et_Terre_Sainte_001.JPG/440px-Musee_de_la_bible_et_Terre_Sainte_001.JPG)
![](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e1/Musee_de_la_bible_et_Terre_Sainte_001.JPG/440px-Musee_de_la_bible_et_Terre_Sainte_001.JPG)
![](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e1/Musee_de_la_bible_et_Terre_Sainte_001.JPG/440px-Musee_de_la_bible_et_Terre_Sainte_001.JPG)
{% endgi %}

### 참조

나무위키 태그[^1]：

나무위키 태그[^2]。

[^1]: 수장형이 좋아할듯
[^2]: 맞지?
