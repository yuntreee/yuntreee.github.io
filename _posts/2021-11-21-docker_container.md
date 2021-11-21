---
title: "[Docker] container"
date: '2021-11-21 13:45:14'
categories: Docker
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 도커 컨테이너란?

이미지로 컨테이너를 생성하면 이미지의 파일과 격리된 시스템 및 네트워크 자원을 사용할 수 있는 독립적인 공간이 생성된다.

이미지는 Image Layer(LowerDir)를 만들고 컨테이너는 Container Layer(UpperDir)을 생성한다. 컨테이너에서 생성되거나 수정되는 내용은 이미지에 영향을 미치지 않는다.



## 2. 도커 컨테이너 상태

![image](https://user-images.githubusercontent.com/60495897/142723379-705a6eb8-2c9a-4c4d-bf39-ec2e8f403a0e.png)



## 3. docker container 명령어

"container"는 docker 명령의 기본값이기 때문에 생략가능하다.

### 1) ps

```bash
# 실행중인 전체 컨테이너 확인
$ docker container ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

|     항목     |                             설명                             |
| :----------: | :----------------------------------------------------------: |
| CONTAINER ID |                      컨테이너의 고유 ID                      |
|    IMAGE     |                    기반이 된 이미지 이름                     |
|   COMMAND    | 컨테이너가 시작될 때 실행되는 명령어 <br> 해당 프로세스는 1번 PID가 된다 |
|   CREATED    |                  컨테이너 생성 후 경과 시간                  |
|    STATUS    | 컨테이너의 상태와 경과 시간 <br> Up: 실행중, Pause: 일시정지, Exited: 종료 |
|    PORTS     |        컨테이너가 개방한 포트와 호스트에 연결된 포트         |
|    NAMES     |                        컨테이너 이름                         |



### 2) create

이미지로 컨테이너를 생성하지만 시작하지는 않는다.

local Repository에 해당 이미지가 없으면 자동으로 docker hub에서 다운로드한다.



### 2) start, stop

컨테이너를 시작 및 종료한다.



### 3) pause, unpause

컨테이너를 일시정지 및 재가동한다.



### 4) run

컨테이너를 생성하고 시작한다. create와 start 명령어를 합친거라고 보면 된다.

|        옵션        |                             설명                             |
| :----------------: | :----------------------------------------------------------: |
| -i (--interactive) | /bin/bash를 통해서 STDIN을 활성화<br> 컨테이너에 attach된 상태라면 STDIN/STDOUT/STDERR 을 사용 가능 <br> 컨테이너에 detach된 상태라면 STDIN 만 사용 가능 |
|     -t (--tty)     |               컨테이너에게 가상 터미널을 할당                |
|    -n (--name)     |                      컨테이너 이름 설정                      |
|   -p (--publish)   |         포트포워딩 <br> <host port>:<container port>         |
| -c (--cpu-shares)  |   cpu 자원 분배 <br> 기본값은 1024이며, 상대적으로 적용됨    |
|   -a (--attach)    |      컨테이너 내부에 접근하여 STDIN/STDOUT/STDERR 사용       |
|   -d (--detach)    |            detach 모드로 실행 (백그라운드로 실행)            |
|     -e (--env)     |                        환경변수 설정                         |
|     --env-file     |                    파일로 환경변수를 설정                    |
|      --expose      | 컨테이너의 포트를 호스트와 연결만 하고 외부에는 노출하지 않음 |
|  -h (--hostname)   |                       호스트네임 설정                        |
|   -m (--memory)    |           메모리 한계 설정 <br> 단위는 b, k, m, g            |
|       --net        | 네트워크 모드 설정 <br> bridge: Docker 네트워크 브릿지에 새 네트워크 생성 <br> none: 네트워크를 사용하지 않음 <br> host: 호스트의 네트워크 네임스페이스를 그대로 사용 |
|     --restart      | 컨테이너 재시작 정책 <br> no: 프로세스가 종료되도 컨테이너를 재시작하지 않음 <br> on-faliure: 종료코드가 0이 아닐 때에만 재시작 <br> on-failure:n : 종료코드가 0이 아닐 때 n회만큼 재시작 <br> always: 항상 재시작 |
|   -v (--volume)    |      도커 볼륨 사용 <br> HostDir:ContainerDir:접근권한       |
|   --volumes-from   |            다른 컨테이너의 도커 볼륨 공유해 사용             |

이미지에 기본으로 설정된 프로세스 말고 원하는 프로세스를 1번 PID로 설정할 수 있다.

```bash
$ docker container run -d --name=ping00 centos /bin/ping localhost
```



### 5) attach

컨테이너 내부에 접근하여 STDIN/STDOUT/STDERR 을 사용한다.



### 6) exec

구동중인 컨테이너에서 새로운 프로세스를 시작한다.

```bash
$ docker container run -d --name=web00 httpd

# 1번 프로세스가 쉘 아님 -> attach해도 쉘 사용 불가능
$ docker container ps -a
CONTAINER ID   IMAGE     COMMAND              CREATED          STATUS         PORTS     NAMES
40a2e9b027fc   httpd     "httpd-foreground"   13 seconds ago   Up 9 seconds   80/tcp    web00

# 컨테이너에서 쉘을 사용하기 위해 별도로 실행
$ docker container exec -it web00 /bin/bash
```



### 7) logs

컨테이너의 1번 PID의 STDIN/STDOUT/STDERR 을 출력한다.

|       옵션       |           설명           |
| :--------------: | :----------------------: |
|  -f (--follow)   |      계속해서 출력       |
| -t (--timestamp) |    타임스탬프도 출력     |
|      --tail      | 마지막 n개의 로그만 출력 |



### 8) stats

작동중인 컨테이너의 실시간 자원 사용률을 보여준다.

```bash
$ docker container stats
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT   MEM %     NET I/O   BLOCK I/O   PIDS
```



### 9) top

작동중인 컨테이너에서 실행되고 있는 프로세스를 확인한다.

이 때 출력되는 PID와 PPID는 HOST PID Namespace를 사용하게 된다.

```bash
# 격리환경에서 1번 PID인 /bin/bash가 1번으로 표시 안됨
$docker container top test
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                2900                2872                0                   04:31               pts/0               00:00:00            /bin/bash
```



### 10) rm

컨테이너를 삭제한다.

|     옵션      |             설명             |
| :-----------: | :--------------------------: |
| -f (--force)  | 컨테이너가 작동중이여도 삭제 |
| -v (--volume) | 컨테이너가 사용한 볼륨 삭제  |

전체 컨테이너를 삭제할 때 ```docker container rm -f $(docker container ps -aq)``` 로 사용한다.



### 11) prune

종료된 컨테이너들을 일괄적으로 삭제한다.



### 12) rename

컨테이너의 이름을 변경한다. 컨테이너ID는 변경되지 않는다.



### 13) cp

컨테이너와 호스트 간의 파일을 복사한다. 컨테이너간의 직접적인 파일 복사는 불가능하다.

```bash
$ docker container cp <Source> <Destination>
```

컨테이너는 ```컨테이너이름:디렉토리``` 형식으로 사용한다.



### 14) inspect

컨테이너의 세부 사항을 출력한다.