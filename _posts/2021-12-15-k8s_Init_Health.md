---
title: "[k8s] Init Container & Health Check"
date: '2021-12-15 01:50:14'
categories: Docker
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. Init Container

### 1) Init Container란?

파드 내부의 앱 컨테이너가 실행되기 전에 초기화를 수행하는 컨테이너다. 

초기화 컨테이너는 여러 개 구성할 수 있으며, 템플릿에 명시된 순서대로 실행된다. 실행에 실패하면 성공할 때까지 재시작되고, 모든 초기화 컨테이너가 실행 완료된 후 메인 컨테이너가 실행된다. 초기화 컨테이너는 실행이 완료되면 삭제된다. 

보안상의 이유로 앱 컨테이너 이미지와 같이 두면 안되는 앱 소스 코드를 별도로 관리할 수 있다.



### 2) Init Container 예제

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
      
  initContainers:
  - name: install
    image: busybox
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - http://info.cern.ch
    volumeMounts:
    - name: workdir
      mountPath: "/work-dir"
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}
```

```shell
$ kubectl exec -it init-demo /bin/bash
```



*workdir* 이라는 볼륨을 만든 후 init container의 */work-dir* 에 마운트한다. 여기에 외부로부터 index.html 을 받아 작성한다.

nginx를 서비스하는 앱 컨테이너는 */usr/share/nginx/html* 에 workdir 볼륨을 마운트한다. 

nginx 이미지에는 없는 소스코드를 init container를 통해 외부에서 받아 사용하는 것이다.



## 2. Health Check

노드의 **kubelet** 이 파드의 health check를 담당한다. 활성 프로브, 준비상태 프로브가 있다.



### 1) Liveness Probe (활성 프로브)

컨테이너의 어플리케이션이 정상적으로 실행중인지 검사한다. 검사에 실패하면 컨테이너가 죽은것으로 판단해 삭제하고 다시 생성한다.



### 2) Readiness Probe (준비 상태 프로브)

어플리케이션의 초기화 과정이 길거나, 다른 어플리케이션이 완료 되어야 정상적으로 서비스를 제공할 수 있는 경우가 있다. 예를 들어, 워드프레스를 사용한 웹서비스는 데이터베이스와 연결된 후에 서비스된다. 이런 경우를 위해 준비 상태 프로브가 사용된다. 

검사에 실패해도 컨테이너를 재가동하지는 않으며, 일정 시간동안 요청 트래픽이 전송되지 않게 한다. 준비가 완료되어 성공 메세지를 전달한 후에 정상적인 서비스가 가능해진다.



### 3) Health Check 예제

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapl
spec:
  containers:
  - name: webapl
    image: maho/webapl:0.1
    # 컨테이너 생성 3초 후 5초마다 http healthcheck 보냄
    # default로 3번 실패하면 컨테이너 재가동 (삭제하고 다시 생성)
    livenessProbe:
      httpGet:
        path: /healthz
        port: 3000
      initialDelaySeconds: 3
      periodSeconds: 5
    
    # 재가동하지는 않음. 파드수준에서 일정시간동안 트래픽을 보내지 않음
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 15
      periodSeconds: 6
```



**httpGet** 은 지정한 포트와 경로로 HTTP GET 요청을 주기적으로 보낸다. HTTP 상태 코드가 2xx 혹은 3xx 이면 성공으로 간주되고, 그 외에는 실패로 간주된다. 

**exec** 은 컨테이너 내에서 명령어를 실행한다. Exit 코드가 0이면 성공으로 간주되고, 그 외에는 실패로 간주된다.

**tcpSocket** 은 지정한 TCP 포트로 연결할 수 있다면 성공으로 간주된다.