---
title: 리눅스에서 무슨무슨 so 파일이 없다고 할 때
date: 2021-03-09 22:22:22
index_img:
category: linux
tags:
  - linux
math: true
mermaid: true
sticky: 100
author: seonjl
---


### 발단
---
nodejs 에서 oracledb 클라이언트를 다운받아 사용하려고 하면 항상 만나는 에러메시지가 있다.

```bash
문제가 있으니
이걸 보고 해결하시오
https://oracle.github.io/node-oracledb/INSTALL.html#overview
```

나는 이러한 에러를 만났다.

```bash
libclntsh.so: cannot open shared object file: No such file or directory
libaio.so.1: cannot open shared object file: No such file or directory
```

so 파일은 shared object 파일이란 뜻으로 특정한 기능을 구현해 놓은 라이브러리 파일이다.

즉 내 컴퓨터에 저 파일이 없어서 오라클 클라이언트를 사용할 수 없다고 하는 문제인 것이다.

파일을 다운로드해서, 리눅스에서 라이브러리를 로딩하려면

***ldconfig*** 라는 명령어를 사용해서, 

/etc/ld.so.conf 파일에 설정된 라이브러리 정보를 
/etc/ld.so.cache 파일로 만들어 준후

리눅스에 로더 가 라이브러리를 찾아 사용할 수 있도록 해주면 된다.

### 해결
---
{% note info %}
💡 리눅스는 so 파일을 아래와 같은 순서로 찾는다

1. system default 경로
2. LD_LIBRARY_PATH
3. binary code 에 hard-coding 된 경로
{% endnote %}

시스템 디폴트 경로에 so 파일을 등록하는 방법을 알아보자.

**1. system default 경로**

이 값은 /etc/ld.so.conf 파일에 설정이 된 값이다. (일반적으로 /usr/local/bin 과 /usr/bin 이다)

```bash
$ more /etc/ld.so.conf
include /etc/ld.so.conf.d/*conf
```

 /etc/ld.so.conf.d/*conf 에 해당하는 모든 파일의 내용을 포함하고 있다.

새로운 라이브러리를 리눅스에 등록하고 싶다면, 
추가할 라이브러리 파일의 경로를 저장한 파일을 
위와 같은 파일 이름 형식으로 만들어 위 경로에 저장해 주면 된다.

```bash
$ ls -al /usr/lib/oracle/19.6/client64/lib
libclntsh.so
lib~~~
lib~~~
lib~~~
```

```bash
# /etc/ld.so.conf.d/oracle.conf
/usr/lib/oracle/19.6/client64/lib
```

원하는 버전의 오라클 클라이언트를 위 경로에 다운 받은 후,

위와 같은 파일을 만들어 경로를 등록 해 준후 ***ldconfig*** 명령어로 등록하면 사용할 수 있다

+) libaio1 은 아래와같이 패키지매니저로 설치해 줄 수 있다. 

```bash
sudo apt-get install libaio1
```

++) 라이브러리 파일에 실행권한이 있냐 없냐 여부는 상관이 없는것 같다.

+++) 라이브러리 파일이 심볼릭 링크인지 여부는 당연히 상관이 없다.



### 참고한 자료들
---
[리눅스에서 무슨무슨 so 파일이 없다고 할 때](https://adnoctum.tistory.com/541)
[C++ - [펌] Linux에서 라이브러리 로딩](https://jacking75.github.io/Linux_lib_setting/)