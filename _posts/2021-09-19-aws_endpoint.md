---
title: "[AWS] 엔드포인트 구성"
date: '2021-09-19 11:30:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. VPC 엔드포인트란

같은 AWS Cloud 내부의 VPC에서 다른 서비스로 네트워크 통신을 하면 트래픽이 퍼블릭 네트워크망으로 나갔다가 다시 목적지로 이동한다. 이는 통신속도를 불필요하게 느리게 하며 보안상으로도 좋지 않다. **VPC 엔드포인트**는 AWS 클라우드 내의 서비스들끼리 안전하게 프라이빗 네트워크 통신을 가능하게 하는 기술이다. 이러한 연결을 **프라이빗 링크** 라고 한다.

VPC 엔드포인트는 4가지 특징을 가진다.

1. 프라이빗 연결을 통해 외부 구간으로 노출이 되지 않는다.
2. 연결 대상 서비스는 동일한 리전에 속한 서비스만 가능하다.
3. 하나의 엔드포인트는 오직 하나의 VPC에만 연결할 수 있다.
4. AWS IAM 기능을 통해 정책을 수립하여 VPC 엔드포인트에 대한 권한 부여가 가능하다.

크게 두가지로 나뉜다.

### 1) 엔드포인트

AWS 퍼블릭 서비스 대상에 대한 프라이빗 연결이다. 

1. 게이트웨이 엔드포인트

   S3와 DynamoDB에 대한 연결이다.

2. 인터페이스 엔드포인트

   S3와 DynamoDB를 제외한 나머지 AWS 퍼블릭 서비스에 대한 연결이다.



### 2) 엔드포인트 서비스

AWS 퍼블릭 서비스 외의 대상에 대한 프라이빗 연결이다. 보통 로드밸런서를 연결하기 위해 사용된다.



## 2. 게이트웨이 엔드포인트 구성

<img src="https://user-images.githubusercontent.com/60495897/133913525-4e71c665-ea6c-400c-85e7-59936cc7b435.png" alt="image" style="zoom:80%;" />

***10.0.0.0/16* VPC** 에 ***10.0.0.0/24* Public 서브넷**과 ***10.0.1.0/24* Private 서브넷**을 생성하여 **S3**에 게이트웨이 엔드포인트로 프라이빗통신 한다.



1. 엔드포인트를 제외한 인프라를 구성한다.

2. S3와 연결되는 게이트웨이 엔드포인트를 생성하고 라우팅 테이블에 추가한다.

   ![image](https://user-images.githubusercontent.com/60495897/133873340-11f6ed2b-42a8-447b-b6a0-ee04eb577793.png)

   ![image](https://user-images.githubusercontent.com/60495897/133873458-a2271155-3aca-4a85-b02b-d9cf9a08c727.png)

엔드포인트 구성 후 프라이빗 서브넷의 인스턴스에 접속하여 **s3.ap-northease-2.amaonaws.com** 에 ping 을 보내면 응답하는 것을 확인할 수 있다.



## 3. 인터페이스 엔드포인트 구성

<img src="https://user-images.githubusercontent.com/60495897/133913547-b32243a2-7243-47ce-87d6-b3986d7f5a2a.png" alt="image" style="zoom:80%;" />

AWS의 **Cloudformation** 서비스에 인터페이스 엔드포인트를 생성하여 프라이빗 통신한다.

기본적으로 AWS 서비스들은 리전별로 DNS 호스트 주소를 가지고 있다. 이러한 AWS 서비스에 인터페이스 엔드포인트를 생성하면 엔드포인트 전용 DNS 호스트가 생성된다. 

인터페이스 엔드포인트를 생성할 때 **프라이빗 DNS 이름 활성화** 라는 항목이 있다.

이를 비활성화하면 기본 DNS 호스트로 접속할 때에는 퍼블릭 네트워크를 통해 서비스에 접속하게 된다.

이를 활성화하면 기본 DNS 호스트로 접속해도 프라이빗 네트워크를 통해 서비스에 접속하게 된다.



따로 게이트웨이가 생성되는 것이 아니라 연결된 서브넷의 IP중 하나를 할당하여 Cloudformation DNS로 연결하는 형태여서 라우팅 테이블에 등록되지 않는다. 



1. 인터페이스 엔드포인트를 생성한다.

   ![image](https://user-images.githubusercontent.com/60495897/133877908-87a7ceab-6aa5-4dfe-95a6-29ed0d429fdf.png)

   차이를 확인하기 위해 프라이빗 DNS를 비활성화하였다.

   ![image](https://user-images.githubusercontent.com/60495897/133877926-c4fefc08-e4de-4bbb-8fc0-902b65986ed6.png)

   필요하다면 보안그룹을 따로 설정하고 엔드포인트를 생성한다.



두개의 서브넷에서 **dig +short** 를 통해 DNS를 입력하면 어디에서 응답하는지 확인한다.

![image](https://user-images.githubusercontent.com/60495897/133878668-387ade6c-c791-4b13-8fa6-662b5230e978.png)

```bash
#========================Public 서브넷========================#
# 기본 DNS로 dig
[ec2-user@ip-10-0-0-52 ~]$ dig +short cloudformation.ap-northeast-2.amazonaws.com
52.95.193.132

# 엔드포인트 전용 DNS로 dig
[ec2-user@ip-10-0-0-52 ~]$ dig +short vpce-05959e83f635a5790-bymlki46.cloudformation.ap-northeast-2.vpce.amazonaws.com
10.0.1.142
10.0.0.115

#========================Private 서브넷========================#
# 엔드포인트 전용 DNS로 dig
[root@ip-10-0-1-109 ~]$ dig +short vpce-05959e83f635a5790-bymlki46.cloudformation.ap-northeast-2.vpce.amazonaws.com
10.0.1.142
10.0.0.115
```

기본 Cloudformation의 DNS로 연결하면 공인IP가 응답하지만, 엔드포인트 전용 DNS로 연결하면 각 서브넷의 사설IP가 응답한다.



## 4. 엔드포인트 서비스 구성

<img src="https://user-images.githubusercontent.com/60495897/133913584-53fafaf6-1d7e-46ed-acd2-65bf5ca0f341.png" alt="image" style="zoom:67%;" />

***10.0.0.0/16* My-VPC**, ***20.0.0.0/16* Custom-VPC** 를 생성하고, 각각 퍼블릭 서브넷을 구성한다. 

My-VPC에는 하나의 인스턴스를, Custom-VPC에는 WEB을 위한 두개의 인스턴스를 구성한다. 

두 웹 호스트를 **NLB**로 묶고 엔드포인트 서비스를 만든다. 

My-VPC에서 인터페이스 엔드포인트를 생성해 엔드포인트 서비스로 연결하여 프라이빗 통신한다.



1. NLB와 엔드포인트를 제외한 인프라를 구성한다

2. Custom-VPC에 WEB1과 WEB2를 타겟그룹으로 한 NLB를 생성한다. 이때, 타겟그룹 프로토콜은 TCP로 설정한다. 구성 후 NLB의 DNS로 접속하면 로드밸런싱이 이루어지고 있는 걸 확인할 수 있다.

3. Custom-VPC에 엔드포인트 서비스를 생성하고 NLB와 연결한다. 

4. My-VPC에 엔드포인트 서비스와 연결되는 인터페이스 엔드포인트를 생성한다.
 
5. 연결 후 엔드포인트의 DNS이름으로 ```curl```을 하면 로드밸런싱이 이루어짐을 확인할 수 있다. ```dig +short <엔드포인트DNS>``` 로 내부통신을 하고 있다는 것을 확인할 수 있다.