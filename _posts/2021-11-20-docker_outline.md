---
title: "[Docker] 컨테이너 서비스 개요"
date: '2021-11-20 18:20:14'
categories: Docker
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 마이크로 서비스 아키텍쳐란?

### 1) 모놀리식(Monolithic) 아키텍쳐

어플리케이션의 모든 기능들이 하나의 프로세스에서 동작한다. 동일한 환경에서 개발되기 때문에 복잡하지 않다.

하지만 기능들이 유기적으로 연결되어 있어 하나의 장애가 전체 서비스의 장애로 연결된다. 또한 한 기능을 업데이트해도 전체 프로세스를 다시 배포해야하므로 업데이트 주기가 길어지게 된다.



### 2) 마이크로서비스(MicroService) 아키텍쳐

기능별로 작은(Micro) 프로세스들을 따로 개발하고 RESTAPI를 통해 서로 통신한다. 모든 기능들이 독립적으로 동작하기 때문에 일부분의 오류가 전체에 영향을 미치지 않는다. 또한 기능별로 개발 및 업데이트가 가능하여 요구사항을 어플리케이션에 빠르게 반영할 수 있다.

하지만 여러 서비스들이 분산되어 있어 관리 및 모니터링이 힘들며, 통신 관련 코드들이 서비스마다 추가되어야 한다.





## 2. 가상머신과 컨테이너

가상화는 서버의 하드웨어 자원을 더 효율적으로 사용하기 위해 등장하였다. 어떤 서비스가 부하가 없는 시간대에 전체 하드웨어 리소스를 독점하기보다, 하나의 서버가 여러개의 서비스를 동시에 제공할 수 있게 한다. 

### 1) 가상머신

서버는 하드웨어/OS/어플리케이션 으로 구성된다. 가상머신은 OS위에 하드웨어 가상화를 해주는 Hypervisor를 두고 그 위에 여러 Guest OS를 설치하여 구동한다. 

이러한 방식은 자원의 오버헤드가 있다. Host OS와 Hypervisor가 많은 리소스를 요구한다. 또한 큰 리소스를 필요로 하지 않는 서비스를 제공할 때에도 OS를 설치해야 하기 때문에 낭비되는 자원이 많다. 

작업은 결국 Host OS를 거쳐야 하기 때문에 여러 계층을 거쳐야 하며, 이는 I/O 성능을 떨어뜨린다.

가상머신들은 서로 격리되어 있지 않다. 여러 가상머신을 올리면 특정 머신이 자원을 독점할 수도 있으며, 보안상에도 문제가 있다.

서비스별로 머신을 개별적으로 구성하고 관리해야 하기 때문에 시스템 관리자의 작업 부하가 증가한다.



### 2) 컨테이너

컨테이너는 Host OS를 논리적으로 격리시키고 하드웨어 자원을 공유하며 구동된다. 

하드웨어 가상화를 거치지 않고 커널 수준의 격리 기술을 제공하기 때문에 I/O 성능이 가상머신보다 월등히 뛰어나다. 구동하기 위해 별도의 OS를 필요로 하지 않고 서비스를 위한 리소스들만 모아 컨테이너가 구성된다. 따라서 최소한의 용량으로 서비스를 운용해 오버헤드가 적고 OS부팅시간이 없다. cgroup을 통해 하드웨어 자원을 할당하기 때문에 하나의 컨테이너가 자원을 독점하지 않는다.

컨테이너는 운영체제를 가상화하기 때문에 독립적인 네임스페이스를 사용하여 각 컨테이너들은 격리된 환경에서 작동한다.

단 하드웨어 자원을 공유하기 때문에 컨테이너들은 동일한 커널을 사용하게 된다.



이러한 컨테이너 기술을 제공하는 대표적인 서비스가 도커(Docker)이다.

![image](https://user-images.githubusercontent.com/60495897/142721128-713a93dd-394c-4edd-8c40-dfd086eb420f.png){: width="70%" height="70%" .align-center}



## 3. 컨테이너 기반 기술

컨테이너는 Linux의 커널수준 기능들을 이용해 구성된다. chroot, namespace, cgroup, overlay 등을 사용한다.



### 1) chroot

chroot를 사용해 컨테이너별로 사용하는 root를 바꿔 격리된 파일 시스템을 제공한다.



### 2) namespace

namespace는 pid, uid, network, hostname 등등의 정보를 저장한 공간이다. 

리눅스는 1번 pid를 가진 프로세스로부터 fork해 새로운 프로세스를 동작한다. 이 때 생성되는 자식 프로세스는 부모 프로세스의 namespace를 공유하게 된다. 별도의 PID tree로 동작되어 호스트나 다른 컨테이너의 프로세스를 볼 수 없어 격리된다.



**Net Namespace**

컨테이너별로 network namespace를 생성해 격리된 네트워크 환경을 구성한다. 컨테이너는 개별적인 인터페이스와 IP 구성을 가지게 된다. 새로운 network namespace는 veth interface를 통해 외부와 통신한다. 

```bash
# network namespace 생성
$ ip netns add testns
$ ip netns
testns


# ip netns exec 명령어로 namespace 내부에 명령어 입력
# 인터페이스 정보 확인
$ ip netns exec testns ip link list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
# 루프백 활성화
$ ip netns exec testns ip link set lo up

# peering으로 연결된 두개의 veth 생성
$ ip link add veth0 type veth peer name veth1
# 5번, 6번 ifindex로 생성됨. 현재 두 veth 모두 host network namespace에 존재
$ ip link list
5: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether e6:78:6b:47:6c:e4 brd ff:ff:ff:ff:ff:ff
6: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 66:ea:4e:88:11:d6 brd ff:ff:ff:ff:ff:ff

# veth1을 testns namespace에 할당
$ ip link set veth1 netns testns
# host network namespace에서 veth1 사라짐. if6번과 연결되었다고 표시됨.
$ ip link list
6: veth0@if6: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
# testns namespace에서 확인
$ ip netns exec testns ip link list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: veth1@if6: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether e6:78:6b:47:6c:e4 brd ff:ff:ff:ff:ff:ff link-netnsid 0

# veth UP
$ ip link set veth0 up
$ ip netns exec testns ip link set veth1 up

# veth IP 할당
$ ip address add 1.1.1.1/24 dev veth0
$ ip netns exec testns ip address add 1.1.1.2/24 dev veth1

# 통신 확인
$ ping 1.1.1.2
```



### 3) cgroup

cgroup은 프로세스 그룹에 대한 CPU, memory, network, storage 자원을 할당 및 제어한다.



### 4) Overlay2

Read Only인 LowDir과 Read Write인 UpperDir로 구성된다. 

모든 변경은 UpperDir에서 일어난다. UpperDir은 투명한 레이어라고 생각하면 된다. 

LowDir에 있던 파일을 수정하면 파일은 UpperDir로 복사된 후 수정된다 (Copy on Write).

LowDir에 있던 파일을 삭제하면 UpperDir에 해당 파일은 삭제되었다는 Metadata가 작성된다. 실제로  파일은 지워지지 않는다.



이 기술은 스토리지를 절약하는데 도움이 된다. 만약 하나의 호스트 위에 동일한 서비스를 제공하는 3개의 어플리케이션을 동작한다고 하자. Overlay를 사용하면 동일한 파일을 3번 저장하는 대신 LowDir에 어플리케이션 구동에 필요한 파일을 작성하고 그 위에 3개의 UpperDir을 만들어 각각의 컨테이너가 사용하면 된다.

그래서 컨테이너에서 LowDir은 ImageLayer, UpperDir은 Container Layer라고 불린다.