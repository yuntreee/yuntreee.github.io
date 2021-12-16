---
title: "[k8s] Deployment"
date: '2021-12-17 01:6:14'
categories: Docker
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. Deployment 란?

디플로이먼트 하위에는 레플리카셋이 있다. 이 레플리카셋을 통해 복제된 파드의 개수를 관리한다. Stateless한 앱을 배포할 때 사용되는 가장 기본적인 컨트롤러이다. 롤아웃/롤백 기능을 지원한다. 디플로이셋은 여러 레플리카셋을 가질 수 있다.

로드밸런싱 기능은 디플로이먼트에 포함되지 않는다. 이 기능은 서비스라는 오브젝트가 담당한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-app
  labels:
    app: web
spec:
  # 복제될 파드의 개수
  replicas: 3
  # 어떤 라벨을 가진 파드를 관리하는지
  selector:
    matchLabels:
      app: web
  
  # 파드 템플릿
  template:
    metadata:
    # selector와 같은 라벨을 요구함
      labels:
        app: web
    spec:
      containers:
      - name: nodejs
        image: nginx
```

```bash
root@master:~# kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
deploy-app-c75bc7665-59vpx   1/1     Running   0          6m14s
deploy-app-c75bc7665-s5pdg   1/1     Running   0          6m14s
deploy-app-c75bc7665-zr2dh   1/1     Running   0          6m14s
```

매니페스트를 실행시키면 3개의 파드가 배포된다. 각 파드의 이름은 디플로이먼트의 name 으로 시작된다.



## 2. Scale In/Out

kubectl 명령어로 replicas 수를 조정해 파드의 개수를 조절할 수 있다.

```bash
root@master:~# kubectl scale deploy deploy-app --replicas=5
deployment.apps/deploy-app scaled
root@master:~# kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
deploy-app-c75bc7665-59vpx   1/1     Running   0          14m
deploy-app-c75bc7665-ddhpr   1/1     Running   0          11s
deploy-app-c75bc7665-n8qqr   1/1     Running   0          11s
deploy-app-c75bc7665-s5pdg   1/1     Running   0          14m
deploy-app-c75bc7665-zr2dh   1/1     Running   0          14m
```

하지만 실제로 이렇게 하지는 않고 서비스를 이용한다. 



## 3. Roll Out

롤아웃은 새로운 이미지를 사용한 컨테이너의 업데이트이다. 

yaml파일을 직접 수정한 후 재배포해도 되고, ```kubectl edit deploy <deploy이름>```을 통해 배포된 deploy의 정보를 수정해도 된다.

롤아웃에는 두가지 정책이 있다.

### 1) Recreate

디플로이먼트를 업데이트하면 이전 버전의 모든 파드가 즉시 종료되고 새로운 버전의 파드가 생성을 시작한다. 따라서 어플리케이션을 완전히 사용할 수 없는 짧은 다운타임이 발생하게 된다.



### 2) RollingUpdate

새로운 파드를 추가하고 오래된 파드를 하나씩 제거하는 방식으로, 업데이트를 하는 중에 서비스를 하는 파드의 개수는 줄어들지 않는다. 파드 수의 상한과 하한을 정할 수 있다. 다운타임은 없지만, 그 대신 이전 버전과 새 버전이 동시에 서비스되는 시간이 있다.

maxSurge는 생성되는 최대 파드 수를 정하며, maxUnavailable은 유지되는 최소 파드 수를 정한다.

예를 들어, 현재 레플리카 값이 10이고 maxSurge값이 25%, maxUnavailable 값은 25%라고 하자. 

최대 25%의 파드를 정지할 수 있으므로, 유지되는 최소 파드 수는 10-10x0.25=7.5 -> 반올림하여 8개가 된다. 

최대 25%만큼의 파드를 초과할 수 있으므로, 생성되는 최대 파드 수는 10+10x0.25=12.5 -> 반올림하여 13개가 된다.

즉, 8~13개의 파드를 유지하며 롤아웃을 진행한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-app
  labels:
    app: web
  annotations:
    kubernetes.io/change-cause: version 1.16.0
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: deploy-nginx
        image: nginx:1.16.0
        ports:
        - containerPort: 80
```

maxUnavailable과 maxSurge의 값은 %말고 정수로도 가능하다.



진행중인 롤아웃을 일시정지/재개를 위해서는 ```kubectl rollout pause/resume``` 명령어를 사용한다.



## 4. RollBack

```kubectl rollout history deploy <deploy이름>``` 명령을 통해 배포이력을 확인할 수 있다.

위에서 배포한 1.16.0 버전의 nginx를 1.17, 1.18로 업데이트 후 배포이력을 확인한다. 확인을 위해 annotation의 change-cause도 함께 수정한다.

```bash
root@master:~# kubectl rollout history deploy deploy-app
deployment.apps/deploy-app
REVISION  CHANGE-CAUSE
1         version 1.17.0
2         version 1.18.0
```

```--revision=숫자``` 옵션을 통해 특정 revision의 이력을 자세히 볼 수 있다.

```bash
root@master:~# kubectl rollout history deploy deploy-app --revision=2
deployment.apps/deploy-app with revision #2
Pod Template:
  Labels:       app=web
        pod-template-hash=6854ffcc87
  Annotations:  kubernetes.io/change-cause: version 1.18.0
  Containers:
   deploy-nginx:
    Image:      nginx:1.18.0
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```

또한 업데이트를 하여도 이전의 replicaset은 지워지지 않고 유지되어있다.

```bash
root@master:~# kubectl get rs
NAME                    DESIRED   CURRENT   READY   AGE
deploy-app-6854ffcc87   4         4         4       5m11s
deploy-app-84488946fc   0         0         0       6m20s
```



저 replicaset을 이용해 원하는 배포버전으로 돌아갈 수 있다.

```kubectl rollout undo deploy <deploy이름>``` 명령은 직전의 replicaset이 동작한다.

```kubectl rollout undo deploy <deploy이름> --to-reivision=숫자``` 명령은 원하는 revision 상태로 돌아간다.



## 5. 배포계획

새로운 이미지로의 배포 계획에는 3가지 방식이 있다.

### 1) 롤링 업데이트

배포된 전체 파드를 한번에 교체하는 것이 아니라 일정 개수 씩 교체하면서 배포한다. deploy의 기본 배포방법이다.



### 2) 블루/그린 (Blue/Green)

기존에 실행됬던 파드 개수와  같은 개수의 파드를 모두 실행한 후 트래픽을 한번에 신규 파드 쪽으로 이전한다. 



### 3) 카나리 (Canary)

버그나 이상이 없는지, 사용자 반응이 어떤지 확인하기 위해 앱 컨테이너 전체를 교체하지 않고 기존 버전을 유지한 채 일부만 새로운 파드로 교체한다.