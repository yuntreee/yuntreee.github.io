---
title: "[AWS] CloudFront"
date: '2021-09-30 02:25:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. CloudFront 란?

### 1) CDN (Contents Delievery Network)

콘텐츠 제공자와 사용자 간 지리적으로 떨어져 있는 환경에서 콘텐츠를 빠르게 제공하기 위한 기술이다. 곳곳에 캐싱서버를 두어, 사용자는 이 캐싱서버로부터 콘텐츠를 제공받을 수 있다. 매번 멀리 있는 오리진 서버에 접속할 필요가 없어 빠른 응답을 받을 수 있으며, 부하분산의 효과도 있다.



### 2) CloudFront

<img src="https://user-images.githubusercontent.com/60495897/135318234-26b37b84-1a80-47c8-bb0f-bd1bc1393ea9.png" alt="image" style="zoom: 65%;" />

AWS가 제공하는 CDN 기술을 CloudFront 라고 한다. 전 세계 리전에 **Edge Location** 이라고 부르는 캐싱 서버들이 동작한다. 

클라이언트는 엣지 로케이션에 콘텐츠 요청을 보낸다. 엣지 로케이션은 **Region Edge Cache**에 데이터가 있는지 확인하고, 없다면 오리진 서버에게 요청한다. 요청받은 데이터는 **distribution**에 의해 요청 리전 엣지 캐시 뿐만 아니라 distribution에 **연결된 모든 리전 엣지 캐시들에게 전송**된다. 

CloudFront는 정적, 동적 콘텐츠에 따라 캐시 동작을 최적화하여 제공한다. 오리진 그룹을 통한 Failover를 유지하며, SSL와 액세스 제어를 지원한다. 



## 2. CloudFront 구성

### 1) 인프라 구성

구성할 인프라는 아래 그림과 같다.

<img src="https://user-images.githubusercontent.com/60495897/135317888-e2eca7b6-845e-4371-81f0-a1151bababd7.png" alt="image" style="zoom:67%;" />



1. CloudFront를 제외한 인프라를 구성한다.

   SA-EC2의 index.html 은 다음과 같이 작성한다.

   ```bash
   $ sudo wget -P /var/www/html/ https://cloudneta.github.io/test.jpg 
   $ sudo vim /var/www/html/index.html
   <head>
   	<link rel='icon' href='data:;base64,iVBORw0KGgo='>
   </head>
   <h1>CloudNet@ CloudFront Test!!</h1><img src='test.jpg'>
   ```



2. Distribution을 생성하기 위해서는 ACM (AWS Certification Manager) 을 통해 인증서를 발급해야한다. ACM 은 **버지니아 북부** 리전에서 사용한다.

   도메인 이름은 **.yuntreee.cf* 이다. DNS검증을 선택 후 *Route 53에서 레코드 생성* 한다. 

   Route 53 호스팅 영역에 아래 그림과 같이 CNAME 레코드가 생성됨을 확인한다.

   ![image](https://user-images.githubusercontent.com/60495897/135314268-dcd456d7-d698-4b26-995f-f48cc59a51ed.png)



3. CloudFront 서비스로 이동하여 Distribution 을 생성한다. 

   원본 도메인은 **오리진의 도메인**을 뜻한다. 생성한 EC2의 Public IPv4 DNS 를 입력한다.

   *기본 캐시 동작* 란은 Region Edge Cache에 관한 설정이다. 기본값으로 둔다.

   *설정* 란은 Distribution에 관한 설정이다. CNAME에 *cdn.yuntreee.cf* 를 추가한다. 이 DNS는 곧 클라이언트가 접속할 때 사용할 주소이다. 기본값 루트 객체란에 */index.html* 을 입력하고 생성한 SSL 인증서를 선택한다.

   

4. Route 53에서 *cdn.yuntreee.cf* 을 Distribution에 Alias 걸어야 한다. 

   <img src="https://user-images.githubusercontent.com/60495897/135316192-c49fbf85-4034-42c0-ba85-9058567b215f.png" alt="image" style="zoom:80%;" />



### 2) 테스트 및 트래픽 흐름

**오리진에 직접 접속**

웹브라우저를 이용해 생성한 오리진에 직접 접속해본다. 

<img src="https://user-images.githubusercontent.com/60495897/135316726-74f2cd8c-c8b2-4ce6-92a9-789c6fc169b4.png" alt="image" style="zoom: 50%;" />

상파울로에 있는 서버에서 10.3MB를 받아오는데 59초가 걸린다.



**CloudFront Distribution에 접속**

이제 **cdn.yuntreee.cf** 로 접속한다. 한번 접속하고 브라우저 캐시를 지운 후 다시 접속해본다.

<img src="https://user-images.githubusercontent.com/60495897/135316976-a34d258f-4ae0-4c47-83ff-f1ddbc64220c.png" alt="image" style="zoom:50%;" />

다운로드하는데 2.5초가 걸린다. 

최초 접속 시 Region Edge Cache에 캐시가 없기 때문에 오리진으로부터 받아온다. 

그 이후의 접속시에는 오리진까지 갈 필요 없이 인접한 Edge Location으로부터 저장된 캐시를 사용해 빠르게 응답받는다.