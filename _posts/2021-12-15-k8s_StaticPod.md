---
title: "[k8s] Static Pod"
date: '2021-12-15 03:12:14'
categories: Docker
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 스태틱 파드란?

파드는 마스터 노드의 kube-apiserver를 통해서 실행된다. 하지만 kube-apiserver도 파드로 실행되는 서버이다. 즉 kube-apiserver를 통하지 않고 실행되는 파드가 있다는 것인데, 이를 Static Pod (스태틱 파드) 라고 한다. 

스태틱 파드는 kubelet이 직접 실행하는 파드로, 이상이 생기면 바로 재시작된다. kubelet은 파드가 아닌 서비스 데몬형태이다. */etc/kubernetes/manifests* 에 기본 스태틱 파드들이 있다. 스태틱 파드는 kube-apiserver, etcd와 같은 시스템 파드를 실행하기 위한 용도로 주요 사용된다. 



## 2. 스태틱 파드 확인

worker노드에서 스태틱 파드를 만들어본다. 

스태틱 파드의 경로는 */var/lib/kubelet/config.yaml* 에 작성되어있다. 

```sh
$ vim /var/lib/kubelet/config.yaml

staticPodPath: /etc/kubernetes/manifests
				-> /etc/static.d 로 변경
```

스태틱 파드 경로를 /etc/static.d 로 변경 후 파드를 하나 작성한다

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  labels:
    app: app
spec:
  containers:
  - image: takytaky/app:v1
    name: app-container
    ports:
    - containerPort: 80
      protocol: TCP
```

```systemctl restart kubelet``` 으로 kubelet을 재시작하고 마스터노드에서 파드를 확인한다.

```shell
root@master:~# kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
app-worker1   1/1     Running   0          46s
```

마스터노드에서 파드를 실행한 적 없는데 app-worker1 이라는 파드가 실행중임을 볼 수 있다. 이는 worker1노드의 kubelet에 의해 실행된 스태틱 파드이다. 

```bash
root@master:~# kubectl delete pod app-worker1
pod "app-worker1" deleted
root@master:~# kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
app-worker1   1/1     Running   0          7s
```

파드를 삭제하여도 다시 재시작됨을 확인할 수 있다.