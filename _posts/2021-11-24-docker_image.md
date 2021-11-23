---
title: "[Docker] image"
date: '2021-11-24 01:05:14'
categories: Docker
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 도커 이미지 관리

### 1) 이미지 형식

도커 이미지는 레지스트리에 보관하며, 레지스트리들은 서버에서 관리된다. 

기본 서버는 *index.docker.io* 이고, 기본 레지스트리는 *library* 이다.

```bash
$ docker info | grep Registry
 Registry: https://index.docker.io/v1/
```

도커 이미지는 ```<서버>/<레포지토리>/<이미지명>:<태그>``` 의 형식을 가진다.



### 2) 이미지 검색/다운로드/삭제

```bash
# 검색
$ docker search <이미지명>
NAME             DESCRIPTION                 STARS     OFFICIAL   AUTOMATED

# 다운로드
$ docker image pull <이미지명>
# 삭제
$ docker image rm <이미지명>
```

*AUTOMATED* 속성은 Dockerfile을 바탕으로 자동 생성된 이미지인지 여부를 보여준다.

태그를 생략 시 제일 최신 버전의 이미지를 다운받는다.

```docker image rm pune -a``` 시에 사용하지 않는 이미지를 전체 삭제한다.



### 3) 이미지 목록/정보/태그

```bash
# 다운받은 이미지 목록 출력
$ docker image ls

# 이미지 상세정보 출력
$ docker image inspect <이미지명>

# 이미지 태그 붙이기
$ docker image tag <이미지명> <태그>
```

태그는 이미지를 새로 만드는 것이 아니라 별명을 붙이는 것이다. 이미지를 레포지토리에 업로드하기 위해 종종 쓰인다.



### 4) 이미지 업로드

이미지를 업로드하려면 이미지 이름을  ```<아이디>/<이미지명>:<태그>``` 형식으로 작성해야 한다. 

아이디는 곧 레포지토리명 이므로, 로그인한 계정의 레포지토리에 이미지가 업로드된다.

```bash
$ docker image tag nginx lewis456/websrv:1.0
$ docker image push lewis456/websrv:1.0
```



## 2. Dockerfile

Dockerfile에는 인프라를 구성하기 위한 요소가 작성되어 있으며, 이를 기반으로 이미지를 생성한다.

Dockerfile에는 다음과 같은 명령이 쓰인다.

| 명령       | 설명               | 명령        | 설명                       |
| ---------- | ------------------ | ----------- | -------------------------- |
| FROM       | 베이스 이미지 지정 | VOLUME      | 볼륨 마운트                |
| RUN        | 명령 실행          | USER        | 사용자 지정                |
| CMD        | 컨테이너 실행 명령 | WORKDIR     | 작업 디렉토리              |
| LABEL      | 라벨 설정          | ARG         | Dockerfile안의 변수        |
| EXPOSE     | 포트 노출          | ONBUILD     | 빌드 완료 후 실행되는 명령 |
| ENV        | 환경 변수          | STOPSIGNAL  | 종료 시그널                |
| ADD        | 파일/디렉토리 추가 | HEALTHCHECK | 컨테이너의 상태 체크       |
| COPY       | 파일 복사          | SHELL       | 기본 쉘                    |
| ENTRYPOINT | 컨테이너 명령      |             |                            |



### 1) 이미지 빌드 과정

각 명령마다 임시컨테이너가 생성되고 변경사항이 적용된 후 이미지를 생성하여 결국에 최종 이미지가 완성되는 형태이다.

예로 들면 다음과 같다.

1. base 이미지로 임시컨테이너가 생성된다. 이 이미지로 이미지 레이어가 생성되고, 그 위에 컨테이너 레이어가 생성된다.

2. 다음 명령에 해당하는 작업이 실행되며, 해당 작업으로 인한 변경사항은 컨테이너 레이어에 작성된다. 작업이 끝나면 이 결과를 *commit* 하여 이미지로 만들고,  이 이미지로 임시 컨테이너를 생성한다.
3. (2)를 반복한다.
4. 최종 명령이 끝나면 *commit*하여 이미지를 생성하고 임시 컨테이너는 삭제한다.



즉, 각 명령별로 이미지 레이어가 생성되어 쌓여가는 과정을 반복한다.

그렇기 때문에 각 명령별로 생성되는 이미지의 사이즈를 최소화해야 한다.

보통 BASE 이미지는 배포용으로 최소한의 리소스만 포함되어 경량화된 이미지를 사용한다.

그리고 빌드 과정에서 생기는 불필요한 파일을 삭제해야 한다. 이 때, 해당 명령이 끝난 후 다음 명령에서 삭제하는 것은 의미가 없다. 이미 이미지에 파일들이 박혀있기 때문에 삭제해도 실제로 삭제되지는 않기 때문이다. 따라서 각 명령에 의해 생성된 파일은 해당 명령 내에서 처리해야 한다.

```bash
$ docker build -t <이미지명> -f <Dockerfile명> <생성할위치>
```





### 2) 명령어 사용법

**FROM**

base 이미지로 컨테이너를 생성한다. 만약 로컬에 없다면 다운로드한다.

```bash
FROM busybox
```



**RUN**

이미지 생성 과정에서 어플리케이션/미들웨어 설치, 환경 설정을 위한 명령 등을 정의한다.

shell 형식과 exec 형식으로 사용된다.

```bash
# shell 형식
RUN apt-get update && apt-get install -y nginx

# exec 형식
RUN ["/bin/bash","-c","apt-get install -y ngnix"]
```

exec형식으로 사용시에는 쉘 환경변수를 사용할 수 없다.



**CMD**

컨테이너가 시작될 때 실행될 명령어로, 1번 프로세스가 된다.

Dockerfile당 하나의 CMD만 작성 가능하다. 여러 개를 기록할 경우 마지막 명령만 유효하다.



**ENTRYPOINT**

CMD와 동일하게 1번 프로세스가 되는 명령어를 기입한다. 

만약 CMD와 ENTRYPOINT가 동시에 작성되었다면, CMD는 ENTRYPOINT 의 인수 형태로 해석된다.

또한, CMD는 컨테이너 실행시에 변경 가능하다.

```bash
# 컨테이너 실행시에 /bin/ping localhost -c 3 이 실행됨
ENTRYPOINT /bin/ping localhost
CMD -c 3

# CMD를 -c 5로 바꿈
$ docker run -it ping00 -c 5
```



**HEALTHCHECK**

컨테이너 안의 프로세스가 정상적으로 작동하고 있는 지 확인한다.

CMD에 이상이 없다면 UNHEALTHY하다고 해서 컨테이너가 종료되는 것은 아니다. 단지 기술한 프로세스의 상태를 체크해준다.

```bash
# 30초마다 index.html을 curl하고 3초 안에 응답을 못받으면 1번 종료코드 반환HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost/index.html || exit 1
```



**ENV**

환경 변수를 설정한다. *run* 시에 --env 명령어로 같은 환경변수를 정의하면 덮어씌워진다.

```bash
# Key value 형식# key 이후의 모든 입력값을 문자열로 처리# 한번에 하나의 변수만 정의ENV myName this is yuntreee# Key=value 형식# 한번에 복수의 변수 정의 가능ENV myName="this is yuntreee" \	myOrder=Pizza\ Pasta\ Salad
```



**EXPOSE**

컨테이너의 공개 포트를 지정한다. *run* 할 때 해당 포트를 호스트의 포트와 매핑할 수 있다.

```bash
EXPOSE 443# 호스트의 8080과 컨테이너의 443을 매핑$ docker run -d -p 8080:443 port00
```



**ADD**

호스트의 파일이나 디렉토리를 이미지에 추가한다.

로컬에서 ADD된 파일은 컨테이너 쉘의 umask 값에 의해서 퍼미션이 결정되지만, 원격지에서 ADD된 파일은 600 퍼미션을 가진다.

```bash
ADD host.html /first_dir/ADD host.html /first_dir/secondfile# 원격지  파일 추가ADD httpsL//github.com/kubernetes/kubernetes/blob/master/LICENSE /fisrt_dir/# 압축/아카이브는 풀어서 추가됨ADD webdata.tar /first_dir/
```



**COPY**

ADD 명령과 비슷하지만, 원격 파일 다운로드나 압축 해제 같은 기능은 없다. 단순히 호스트의 파일이나 디렉토리를 이미지로 복사한다.



**VOLUME**

지정한 이름의 마운트 포인트를 생성하고, 호스트나 이외의 컨테이너로부터 마운트한다.

```bash
VOLUME /container# /container 디렉토리를 마운트 포인트로 잡음# /var/lib/docker/volumes 에 마운트할 디렉토리 생성됨
```

