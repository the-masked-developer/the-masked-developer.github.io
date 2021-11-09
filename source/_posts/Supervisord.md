---
title: Supervisord 찍어먹기
toc: true
date: 2021-11-09 23:05:19
tags:
category: 프로그래밍
author: 이민혁
---

## Supervisor

supervisord 또는 supervisordctl command 는 기본적으로 다음과 같은 경로에서 순서대로 `supervisord.conf` 을 찾는다.

- ../etc/supervisord.conf (Relative to the executable)
- ../supervisord.conf (Relative to the executable)
- $CWD/supervisord.conf
- $CWD/etc/supervisord.conf
- /etc/supervisord.conf
- /etc/supervisor/supervisord.conf (since Supervisor 3.3.0)

우선 다음과 같이 예제 어플리케이션을 생성했다.
```python
#!/usr/bin/env python3
from time import sleep


print("Starting application...")

for i in range(5):
    sleep(1)
    print(f"[{i}] working...")
```

```yaml
# docker-compose.yml
version: "3.8"

services:
  app:
    image: python:latest
    volumes:
      - "./app.py:/app.py"
    command: /app.py
```

실행 할 경우.
```
~/Dropbox/box/demos/supervisord master* ⇣⇡ ❯ dc up
Starting supervisord_app_1 ... done
Attaching to supervisord_app_1
app_1  | Starting application...
app_1  | [0] working...
app_1  | [1] working...
app_1  | [2] working...
app_1  | [3] working...
app_1  | [4] working...
supervisord_app_1 exited with code 0
```
작업을 마친 후 자동으로 종료된다.

이제 supervisord.conf 를 생성한다. 파일 형식은 윈도우즈의 `ini` 로써 각 섹션은 `[HEADER]` 로 나타내며 
섹션안에 `Key=Value` 페어가 들어간다. 공식 홈페이지를 보고 처음부터 한땀한땀 해도 되겠지만 공식 레포의 [샘플](https://github.com/Supervisor/supervisor/blob/master/supervisor/skel/sample.conf) 로 시작하는게 좋을듯 하다. 
(set filetype=dosini 로 문법 하이라이팅을 켤 수 있다.)

![supervisord.conf](https://minhyeoky.github.io/posts/supervisor/img/supervisord.conf.png)
- HTTP 서버를 돌릴게 아니므로, `unix_http_server` 및 `inet_http_server` 는 지운다.
- `supervisord` 는 supervisord 프로세스에 대한 설정이므로 냅둔다.
- rpc 또한 해당사항이 없으므로 `rpcinterface` 도 지운다.
- `supervisorctl` 의 경우 쉘 커맨드 설정을 위함인데, 유저별로 권한을 주는 등 설정할 필요는 없으므로 지운다.
- `include` 는 다른 supervisord.conf 파일을 포함시킬 수 있게 해준다. 지운다.
- `group` 은 program 들을 묶을 수 있게 해준다. 지운다.
- `eventlistener` 는 supervisor’s event system. 로부터 이벤트를 받을 수 있는 프로그램이다. 지운다.

supervisor 는 python 으로 구현되어 있으므로 pip 로 설치한다. 그러므로 Dockerfile 을 생성하고 
설치하도록 했다. (버퍼를 꺼줘야 컨테이너 로그가 바로 보인다.)
```dockerfile
FROM python:latest

ENV PYTHONUNBUFFERED 1

RUN pip install supervisor
```

supervisor.conf 파일을 docker-compose.yml 의 volume 에 추가한다.

```yaml
services:
  app:
    build: ./
    volumes:
      - "./app.py:/app.py"
      - "./supervisord.conf:/etc/supervisord.conf"
    command: supervisord

```
이렇게 실행 할 경우 nodaemon 값을 true 로 해줘야한다.

```dosini
[program:app]
  command=/app.py              ; the program (relative uses PATH, can take args)
```
                                                                
이제 다시 실행할 경우 app 의 로그는 보이지 않지만 정상적으로 종료되었음을 알 수 있다. 
```
app_1  | /usr/lib/python3/dist-packages/supervisor/options.py:474: UserWarning: Supervisord is running as root and it is searching for its configuration file in default locations (including its current working directory); you probably want to specify a "-c" argument specifying an absolute path to a configuration file for improved security.
app_1  |   self.warnings.warn(
app_1  | 2021-09-01 10:01:47,610 CRIT Supervisor is running as root.  Privileges were not dropped because no user is specified in the config file.  If you intend to run as root, you can set user=root in the config file to avoid this message.
app_1  | 2021-09-01 10:01:47,613 INFO supervisord started with pid 1
app_1  | 2021-09-01 10:01:48,616 INFO spawned: 'app' with pid 8
app_1  | 2021-09-01 10:01:49,707 INFO success: app entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
app_1  | 2021-09-01 10:01:51,714 INFO exited: app (exit status 0; expected)
```

로그는 stdout_logfile 을 통해서 std 또는 file 에 쓰면 된다.

```dosini
[supervisord]
  stdout_logfile=/dev/stdout        ; stdout log path, NONE for none; default AUTO
  stdout_logfile_maxbytes=0   ; max # logfile bytes b4 rotation (default 50MB)
```

program 의 auturestart 의 기본값은 unexpected 로써 exit code 0 이 아닌 경우 재시작한다. 그러므로 
app.py 에서 sys.exit(1) 로 비정상 종료 코드를 내보낸다면 supervisord 가 프로세스를 재시작해준다.

```
app_1  | 2021-09-01 12:09:11,750 INFO spawned: 'app' with pid 1819
app_1  | Starting application...
app_1  | [0] working...
^[app_1  | 2021-09-01 12:09:12,783 INFO success: app entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
app_1  | [1] working...
app_1  | [2] working...

app_1  | 2021-09-01 12:09:14,862 INFO exited: app (exit status 1; not expected)
app_1  | 2021-09-01 12:09:15,869 INFO spawned: 'app' with pid 1820
```


