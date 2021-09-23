---
title: "[AWS] NAT 인스턴스 구성"
date: '2021-09-23 23:14:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. NAT 디바이스란?

AWS에서 NAT 디바이스에는 **NAT게이트웨이 ** 와 **NAT 인스턴스** 를 말한다. 프라이빗 서브넷에 배치된 인스턴스는 공인IP가 없어 NAT를 이용해 외부 네트워크 또는 AWS 서비스들과 연결한다.

NAT 게이트웨이는 AWS에서 제공하는 서비스이다. 5Gbps ~ 45Gbps로 필요에 따라 대역폭을 자동으로 조절하며, 단일 상대에 대해 분당 최대 55,000개의 동시연결을 지원한다. **NAT 게이트웨이는 외부와 통신이 가능한 Public 서브넷에 위치해야 한다.**

NAT 인스턴스는 말 그대로 인스턴스를 NAT 동작하게 하여 사용하는 것이다. 사용자가 직접 관리하며, 보안그룹 연결 및 SSH 접속이 가능하다.



## 2. NAT 인스턴스 구성

구조는 다음과 같다.

![image](https://user-images.githubusercontent.com/60495897/134542132-f699c00f-00ab-4530-a506-4fa0378e01db.png)

Private 인스턴스가 외부 네트워크와 통신 시 패킷을 NAT 인스턴스에게 라우팅하고, NAT 인스턴스는 PAT동작을 한다.



1. 위 그림대로 인프라를 구성한다. NAT 인스턴스는 *amzn-ami-vpc-nat* 가 포함된 AMI를 사용한다.

2. NAT 인스턴스는 **작업->네트워킹->소스/대상 확인 변경** 에서 비활성화해준다. 

3. NAT 인스턴스에 접속하여 다음과 같은 사항을 확인한다.

   ```bash
   # 포워딩 활성화를 확인
   $ sudo cat /proc/sys/net/ipv4/ip_forward
   1
   
   # iptable에서 NAT 동작 확인
   $ sudo iptables -nL POSTROUTING -t nat -v
   Chain POSTROUTING (policy ACCEPT 1 packets, 84 bytes)
    pkts bytes target     prot opt in     out     source               destination                                                                                                              
     131  8489 MASQUERADE  all  --  *      eth0    0.0.0.0/0            0.0.0.0/0 
   ```



Private 인스턴스에 접속하여 외부로 ping을 보내면 잘 동작하는 것을 확인할 수 있다.

```curl http://checkip.amazonaws.com``` 을 입력하면 NAT 인스턴스의 공인IP가 응답된다. 