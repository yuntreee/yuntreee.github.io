---
title: "[AWS] Public/Private 서브넷 구성"
date: '2021-09-17 01:23:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 구성

![image](https://user-images.githubusercontent.com/60495897/133650656-f5b17227-ad51-432f-917c-f411b8baa695.png)

***10.0.0.0/16* VPC**에 ***10.0.0.0/24* Public 서브넷**과 ***10.0.1.0/24* Private 서브넷**을 구성한다.

Public 서브넷의 라우팅 테이블은 외부와 통신하기 위해 **IGW**에 연결하고

Private 서브넷의 라우팅 테이블은 **NAT GW**에 연결한다.



## 2. Public 서브넷 구성

1) VPC를 생성한다.

   ![image](https://user-images.githubusercontent.com/60495897/133643153-193a46fa-ec54-4272-a56b-8e3220ff208e.png)



2) 서브넷을 생성한다.

   ![image](https://user-images.githubusercontent.com/60495897/133644034-5ebdfd29-b85b-4f8e-a492-2b0240f0136e.png)



3) IGW를 생성하고 VPC에 Attatch 한다.

   ![image](https://user-images.githubusercontent.com/60495897/133643429-127d9806-ba00-4999-bdfb-632c3c43dcaa.png)

   ![image](https://user-images.githubusercontent.com/60495897/133643653-e2f237cf-54ea-4499-9956-8a534403cbec.png)



4) Public 라우팅 테이블을 생성하고 IGW로 경로설정 후 서브넷에 연결한다.

   ![image](https://user-images.githubusercontent.com/60495897/133644567-a708a53c-9074-4fb4-a86c-bc55d3d8ca67.png)

   ![image-20210917005249654](C:\Users\lewis\AppData\Roaming\Typora\typora-user-images\image-20210917005249654.png)

   ![image](https://user-images.githubusercontent.com/60495897/133644813-e389525c-d5da-4680-a466-db17ad973368.png)



6) Public 서브넷에 EC2 인스턴스를 생성한다.

   ![image](https://user-images.githubusercontent.com/60495897/133645172-2be95001-b571-469f-8029-b75bcc3df233.png)



## 3. Private 서브넷 구성

1) 서브넷을 생성한다. 이 때, 가용영역은 Public 서브넷과 다르게 설정한다.

   ![image](https://user-images.githubusercontent.com/60495897/133648608-c5aa86fa-fbc1-465e-a8d9-5f72b6d36e10.png)



2) Public 서브넷에 NAT GW를 생성한다. Public 서브넷에 생성해야 Private 서브넷이 NAT를 통해 외부와 통신할 수 있다.

   ![image](https://user-images.githubusercontent.com/60495897/133646121-59a96010-59fd-479a-b0cd-ce3c09523398.png)



3) Private 라우팅 테이블을 생성하고 NAT로 경로설정 후 서브넷에 연결한다.

   ![image](https://user-images.githubusercontent.com/60495897/133646556-5f2383bd-eb46-4af3-9e0e-e3dce7361967.png)

   ![image](https://user-images.githubusercontent.com/60495897/133646785-97ba6a15-574a-43cf-98b2-8d5e4706b45f.png)

   ![image](https://user-images.githubusercontent.com/60495897/133646867-67e96747-cee3-4173-84a0-380ac57c9f3b.png)



4) Private 서브넷에 EC2를 생성한다. 퍼블릭 IP 할당은 비활성화하고, 사용자 데이터에 아래와 같은 쉘 스크립트가 실행되도록 한다.

   ![image](https://user-images.githubusercontent.com/60495897/133649013-defb3a22-85e8-47ce-9284-00ea00ec81b4.png)

   ```bash
   #! /bin/bash 
   ( echo "qwe123" & echo "qwe123" ) | passwd --stdin root 
   
   sed -i "s/^PasswordAuthentication no/PasswordAuthentication yes/g" /etc/ssh/sshd_config 
   
   sed -i "s/^#PermitRootLogin yes/PermitRootLogin yes/g" /etc/ssh/sshd_config
   
   service sshd restart
   ```

   

Public 서브넷에 생성된 EC2는 공인IP가 없고 사설IP만 있기 때문에 외부에서 직접 접근할 수 없다. 

