---
title: "[Docker] network"
date: '2021-11-23 00:20:14'
categories: Docker
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 도커 네트워크

### 1) Default Bridge

도커를 설치하면 기본값으로 **docker0** 브릿지가 생성된다. 

컨테이너를 생성하면 **컨테이너 내부에 eth** 과 **호스트에 veth**이 생성되며, 둘은 서로 연결되어 있다.

docker0 브릿지는 veth과 호스트의 물리적 eth을 연결해주는 역할을 한다.

컨테이너들은 브릿지의 IP대역대에서 낮은 수부터 차례대로 IP를 할당받는다.

컨테이너들이 외부와 통신하기 위해서는 **포트 포워딩**이 사용된다. 호스트의 물리적 네트워크 인터페이스로 요청이 들어오면 패킷의 포트를 보고 매칭되는 컨테이너로 브릿지를 통해 전송한다.

```bash
$ ip addr
# Host의 물리적 NIC
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
# 도커의 가상 Bridge
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> ...
# 컨테이너와 연결된 veth
13: veth02d81e5@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
```



호스트의 8080 포트로 들어오는 패킷이 컨테이너의 80포트로 들어오게 설정한다.

```bash
$ docker run -d --name=net00 -p 8080:80 nginx
$ iptables -t nat -vnL
Chain POSTROUTING (policy ACCEPT 131 packets, 7944 bytes)
 	0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
    0     0 MASQUERADE  tcp  --  *      *       172.17.0.3           172.17.0.3           tcp dpt:80

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
    0     0 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.3:80
    
$ docker inspect net00
            "Ports": {
                "80/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "8080"
                    },
                    {
                        "HostIp": "::",
                        "HostPort": "8080"
                    }
                ]
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "f6494a8e8b4ea666f73735d2c2b4f4c0875d77380cadf7278eaeacbba929d521",
                    "EndpointID": "1b4f162c29da8fd691017011e2fc8bbedeeca4105e5bd6dbe99d142ac3975547",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.3",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
```



![image](https://user-images.githubusercontent.com/60495897/142878951-9d50903d-521b-4c08-b847-ce8e42d13ffd.png)



### 2) Custom Bridge

**docker0** 이 아닌 새로운 브릿지를 생성하여 사용할 수도 있다.

```bash
$ docker network create --driver=bridge --subnet=172.20.0.0/16 custombridge
$ docker network ls
NETWORK ID     NAME           DRIVER    SCOPE
4d9ae769a19b   custombridge   bridge    local

$ docker run -d --name=net01 --network=custombridge nginx
$ docker inspect net01 | grep -A 15 "Networks"
            "Networks": {
                "custombridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": [
                        "cc11f3061f3a"
                    ],
                    "NetworkID": "4d9ae769a19b89bf7c621fcbbecf9f058542a8997982178117050c9f8c1addd3",
                    "EndpointID": "ce5fc278456120c01c82ac042a2f580868036ef6fa8764f8f331526c82abdf7b",
                    "Gateway": "172.20.0.1",
                    "IPAddress": "172.20.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:14:00:02",
```



### 3) Host Bridge

호스트의 네트워크 환경을 그대로 사용해 별도의 포트포워딩 없이 통신한다.

하지만  호스트 네트워크와 격리되어 있지 않기 때문에 보안상 좋지 않다.

```bash
$ docker run -itd --name=test00 --net=host centos
```



### 4) None Network

```docker network ls```를 하면 Name이 none인 네트워크가 있다. 이는 컨테이너와 외부의 통신을 사용하지 않게 한다.

```bash
$ docker run -itd --name=non00 --net=none centos
```



### 5) Container Network

다른 컨테이너의 NET namespace를 공유한다. 두 컨테이너는 같은 IP와 MAC을 갖게 된다.

```bash
$ docker run -itd --name=origin centos
$ docker run -itd --name=other --net=container:origin centos
```

이는 컨테이너의 IP를 고정시킬 때 사용된다. 

IP만 제공해줄 *Infrastructure 컨테이너* 를 하나 생성하고, 서비스할 컨테이너는 Infrastructure 컨테이너의 네트워크 네임스페이스를 사용한다. 이 컨테이너는 종료되었다가 다시 실행되어도 Infrastructure의 컨테이너 IP 를 사용하기 때문에 변하지 않는다.



### 6) 컨테이너간 통신

``--link <컨테이너이름>``를 사용하면 컨테이너의 이름과 IP가 */etc/hosts*에 등록되어 사용할 수 있다. 컨테이너의 IP가 변경되면 */etc/hosts*의 내용도 변경된다.

```bash
$ docker run -itd --name=db00 centos
$ docker run -it --name=web00 --link=db00 centos
[root@b3573b2cbda2 /]# cat /etc/hosts
127.0.0.1       localhost
172.17.0.2      db00 43b93102ec03
172.17.0.3      b3573b2cbda2
```



### 7) 부하분산

완벽한 부하분산은 아니지만, Round Robin 알고리즘을 통해 여러 컨테이너를 돌아가며 연결되게 할 수 있다.

``--netalias`` 는 Custom Bridge 에서만 사용 가능하다.

도커 엔진에 내장된 DNS가 설정한 호스트네임을 각 컨테이너의 주소로 변환한다.

```bash
# 여러 컨테이너와 연결된 rr 이라는 호스트네임 생성
# 컨테이너들은 동일한 브릿지 사용
$ docker run -itd --name=test00 --net=custombridge --net-alias=rr centos
$ docker run -itd --name=test01 --net=custombridge --net-alias=rr centos

$ docker run -it --name=ping00 --net=custombridge centos
[root@b3573b2cbda2 /]# ping -c 1 rr
```
