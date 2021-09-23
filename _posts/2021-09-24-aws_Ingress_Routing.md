---
title: "[AWS] VPC Ingress Routing 구성"
date: '2021-09-24 01:30:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. VPC Ingress Routing 이란?

퍼블릭 서브넷 내의 인스턴스들이 보안을 위해 공인IP로 외부와 직접 통신하는 것을 원치 않을 때가 있다. VPC Ingress Routing은 미들박스 인스턴스를 두고, NAT 동작 대신 **라우팅을 통해** 외부와 연결하는 다리 역할을 하는 구조이다. 

이러한 구조에서 **NAT는 한번만** (IGW 혹은 VGW에서) 일어나게 되고, 미들박스를 통해 **AWS의 모니터링, 보안 관리 및 운영**에 용이하다.



## 2. VPC Ingress Routing 구성

구조는 다음과 같다.

![image](https://user-images.githubusercontent.com/60495897/134526639-ff80e102-315a-426a-a279-1e7bbcc8cdc5.png)

Inner Public 서브넷은 외부로 나가는 트래픽을 미들박스에게 보내고, IGW는 외부에서 Inner 퍼블릭 서브넷으로 들어오는 트래픽을 미들박스에게 보낸다.

즉, Inner Public 서브넷은 외부와 통신하기 위해 DMZ Public 서브넷의 미들박스를 거치게 된다. 



1. 위 그림대로 인프라를 구성한다. 보안그룹은 모든 주소로부터 인바운드되는 SSH, HTTP(S), ICMP를 허용한다.

   IGW는 RT와 엣지연결한다.

   

2. 미들박스 인스턴스는 **작업->네트워킹->소스/대상 확인 변경** 에서 비활성화해준다. 

   인스턴스는 원래 종단장비여서 패킷의 src/dst가 자신이 아니면 drop한다. 

   이 기능을 비활성화 하여인스턴스가 종단장비가 아닌 중계기 역할을 하도록 해준다. 

3. 미들박스 인스턴스에 접속하여 다음 두 명령어로 커널 정보를 변경한다.

   ```bash
   # 포워딩 기능을 활성화
   $ sudo sysctl -w net.ipv4.ip_forward=1
   
   # ICMP 리다이렉션 기능을 비활성화
   $ sudo sysctl -w net.ipv4.conf.eth0.send_redirects=0
   ```

   

4. Inner Public 서브넷의 인스턴스에 웹서버를 가동하고, 미들박스에서 ```sudo iptraf-ng``` 로 트래픽을 모니터링해본다.

   ![image](https://user-images.githubusercontent.com/60495897/134537645-56be8704-799f-408d-b724-acaf6595e005.png)

   웹서버에 접속하면 패킷이 미들박스를 한번 거쳐가는 것을 확인할 수 있다.



