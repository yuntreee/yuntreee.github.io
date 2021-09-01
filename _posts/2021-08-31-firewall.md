---
title: "[CentOS 8] 방화벽 (Firewall)"
date: '2021-08-31 00:26:14'
categories: Linux
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 방화벽 (Firewall) 이란

일반적으로 내부 네트워크와 외부 네트워크의 경계 지점에 위치한다. 외부 네트워크의 침입으로부터 내부 네트워크를 보호하기 위해 사용되는 정책이다. 서비스를 요구한 클라이언트의 IP, 포트번호, 사용자 인증에 기반을 두고 외부 침입을 차단한다. 

방화벽은 기본적으로 들어오고 나가는 패킷에 대해 지정된 정책과 규칙을 사용하여 허용(accept)과 차단(reject) 라는 행동을 통해 모든 패킷을 통제하게 된다.

CentOS 8 에서는 firewalld 데몬이 작동한다. firewall-cmd 나 firewall-gui를 통해 설정한 내용을 firewalld 데몬이 iptables로 전달한다.



## 2. 방화벽 관리

### 1) Zone 이란

사전 정의된 Zone 이란 것이 있으며, 트래픽의 종류에 따른 규칙이다.

기본적으로 방화벽은 외부에서 들어오는 패킷을 통제하기 위해 존재하기 때문에 drop을 제외한 zone들은 나가는 패킷에 대한 통제를 하지 않는다.

|    Zone 이름     |                             설명                             |
| :--------------: | :----------------------------------------------------------: |
|     trusted      |              들어오고 나가는 모든 트래픽을 허용              |
|       home       | ssh, mdns, ipp-client, samba-client, dhcpv6-client를 제외한 모든 들어오는 패킷 차단 |
|     internal     |                     home 과 규칙이 동일                      |
|       work       | ssh, ipp-client, dhcpv6-client를 제외한 모든 들어오는 패킷 차단 |
| public (default) |    ssh, dhcpv6-client를 제외한 모든 들어오는 패킷을 차단     |
|     external     | ssh 를 제외한 모든 들어오는 패킷을 차단. 이 zone을 통해 나가는 ipv4 트래픽은 마스커레이드됨. |
|       dmz        |               ssh를 제외한 모든 트래픽을 차단                |
|       drop       |              들어오고 나가는 모든 트래픽을 차단              |



```--get-default-zone``` : 현재 default zone을 확인한다.

```--set-default-zone=<ZONE>``` : default zone을 설정한다.

```--get-zones``` : 사용 가능한 모든 zone을 출력한다.

```--list-all-zones``` : 사용 가능한 모든 zone과 자세한 정보를 출력한다.

```--get-active-zone``` : 현재 활성화된 zone 을 출력한다.



### 2) Zone에 규칙 추가

특정 zone에 대해 적용하고 싶다면 모든 옵션 뒤에 ```--zone=<ZONE>``` 을 붙이면 된다. 생략시 default zone 에 적용된다.

영구적용 시 ```--permanent``` 옵션을 붙인다. 설정 후 ```--reload``` 옵션으로 firewalld를 재실행해야 적용된다.

```--add``` 의 반대 명령어는 ```--remove``` 이다.



```--add-source=<CIDR>``` : zone에 특정 ip 추가

```--add-interface=<Interface>``` : zone에 인터페이스 추가

```--add-service=<Service>``` : zone에 서비스 추가

```--add-port=<Port>``` : zone에 포트 추가

