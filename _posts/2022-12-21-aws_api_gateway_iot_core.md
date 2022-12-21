---
title: "[AWS] API Gateway를 사용해 IoT Core 메시지 Publish 하기"
date: "2022-12-21 15:230:30"
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---

## 1. 개요

AWS IoT Core는 MQTT와 HTTPS 방식을 지원한다.

IoT 디바이스에서 MQTT로 메세지를 보낼 수 있다면 좋겠지만, 만약 그럴 수 없는 경우 IoT Core가 제공하는 HTTP URL를 사용해 메세지를 POST할 수 있다.

단, HTTPS 통신은 Publish 만을 지원한다. Subscribe는 MQTT를 사용해야한다.

이 글에서는 디바이스가 **이미 정의된 REST API URL**을 통해 메세지를 전송하는데, 이를 API Gateway를 사용해 AWS IoT Core로 전송하는 방법을 설명한다.

![image](https://user-images.githubusercontent.com/60495897/208835472-efa1c769-f6d9-41b9-a6c1-14aa67c6b633.png){: width="90%" height="90%" .align-center}

## 2. 설정

### 1) API Gateway 설정

![image](https://user-images.githubusercontent.com/60495897/208826671-2533240a-59f7-4f51-b545-b8de7cede599.png){: width="90%" height="90%" .align-center}

AWS API Gateway -> REST API 를 생성한다.



![image](https://user-images.githubusercontent.com/60495897/208828773-a956c0dc-50c4-42fe-8d2d-4d6e7cb3d2fa.png){: width="90%" height="90%" .align-center}

POST 메소드를 생성한다.

**Integretion Type : AWS Service**

**AWS Region : ap-northeast-2**

**AWS Service : IoT Data**

**AWS Subdomain : 계정의 IoT Core Endpoint 앞부분 (xxxxx-ats)**

**HTTP method : POST**

**Path override : /topics/{메세지를 Publish 할 토픽}**

**Execution role : 해당 토픽에 Publish 할 수 있는 권한을 가진 역할**

**Content Handling : Passthrough**



![image](https://user-images.githubusercontent.com/60495897/208829390-2dc12c26-721c-4f36-b278-2d59cb0dfca6.png){: width="90%" height="90%" .align-center}

메소드 생성 후 API를 배포한다.



### 2) 테스트

![image](https://user-images.githubusercontent.com/60495897/208833013-0797cd8f-17a0-481e-8c08-6d6f4f14c064.png){: width="90%" height="90%" .align-center}

Postman 에서 Body에 메세지를 넣고 POST 호출하면 "OK" 리스폰스를 받고,

IoT Core -> MQTT test client를 보면 메세지가 보내진 것을 확인할 수 있다.



---

[참고문서]

https://docs.aws.amazon.com/iot/latest/apireference/API_iotdata_Publish.html

https://docs.aws.amazon.com/solutions/latest/constructs/aws-apigateway-iot.html
