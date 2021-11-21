---
title: "[Docker] volume"
date: '2021-11-21 15:15:14'
categories: Docker
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 도커 볼륨이란?

컨테이너가 삭제되면 컨테이너 레이어도 함께 삭제되어 다른 프로세스나 컨테이너가 해당 데이터를 필요로 하는 경우 사용할 수 없게 된다. 

도커 볼륨은 호스트의 파일시스템을 컨테이너에 마운트시켜 컨테이너가 삭제되어도 데이터는 남아있게 한다.



## 2. docker volume 명령어

`run`할 때 `-v` 옵션을 주어 컨테이너 생성시에 **bind mounts** 생성이 가능하지만 호스트의 파일시스템 디렉토리 구조에 의존적이다.

```bash
$ docker container run -v /tmp/hostdir:/container_dir:ro --name teste centos
```

인자에 컨테이너에서 마운트포인트만 명시하면 */var/lib/docker/volumes* 에 무작위 16진수로 된 디렉토리가 생겨 이걸 마운트한다.

```bash
$ docker inspect test | grep -A 10 "Mounts"
        "Mounts": [
            {
                "Type": "volume",
                "Name": "adf2a33dae1232afebda928df27f291ecb664d2e160a9da2968922bd3f73ccce",
                "Source": "/var/lib/docker/volumes/adf2a33dae1232afebda928df27f291ecb664d2e160a9da2968922bd3f73ccce/_data",
                "Destination": "/container_dir",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }

```



``docker volume`` 을 사용하면 호스트에 물리적으로 연결되어있는 파일시스템에 마운트 가능하며, NFS와 같이 공유된 파일시스템에 마운트하는 것도 가능하다. 볼륨을 어떤 컨테이너가 사용하는지, 어떻게 마운트 되는지 관리하기 용이하다.

도커 볼륨파일은 */var/lib/docker/volumes* 에 위치한다.



```bash
# 도커 볼륨 생성
$ docker volume create testvol
$ docker volume ls
DRIVER    VOLUME NAME
local     testvol

$ echo "Hello" > /var/lib/docker/volumes/testvol/_data/f1

$ docker run -it -v testvol:/container --name=test00 centos
[root@93adbdb9e03f /]# cat /container/f1
Hello
```



`--volumes-from` 옵션을 사용하면 다른 컨테이너에서 사용하는 볼륨을 동일하게 사용할 수 있다.

```bash
$ docker run -it --volumes-from test00 --name test01 centos
[root@9774cc3bda9a /]# ls /container/
f1
```

