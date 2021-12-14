---
title: "[AWS] EC2 Auto Scaling (2)"
date: '2021-11-04 00:17:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 대상 추적 조정 정책

![image](https://user-images.githubusercontent.com/60495897/140084810-fc601fda-427e-4aa2-8ac6-b8e72e583b3e.png)

서로 다른 두개의 서브넷에 Auto Scaling 그룹을 만들고, 이를 ALB에 연결한다. 인스턴스는 웹 서비스를 제공한다.



1. Auto Scaling을 제외한 인프라를 구성한다.

   

2. Auto Scaling 시작 구성을 생성한다. 사용자 데이터에 웹 서버가 설치되도록 쉘 스크립트를 작성한다.

   ```bash
   # 웹 부하를 줄 수 있는 기능이 설정된 파일 다운로드
   #!/bin/bash
   amazon-linux-extras install -y epel
   yum install -y httpd php git stress
   git clone https://github.com/suminhong/cloudformation_yaml.git /cf
   mv /cf/webstress.php /var/www/html/index.php
   rm -f /var/www/html/index.html
   systemctl enable httpd
   systemctl start httpd
   ```

   

3. 시작 구성을 사용하여 Auto Scaling 그룹을 생성한다. 가용 구역 a와 c를 선택하고 ALB에 연결한다. 

   용량을 설정하고, CPU 평균 사용율 지표를 사용한 대상 추적 조정 정책을 설정한다.

   CloudWatch에 의해 모니터링되는 인스턴스들의 CPU 평균 사용량이 설정한 수치를 넘어가면 트리거가 작동해 Scale Out 된다.



Auto Scaling 그룹 생성을 마치면 설정한 용량대로 초기 인스턴스가 생성된다. 

![image](https://user-images.githubusercontent.com/60495897/140091478-189080a9-ce0e-4f3e-a47b-4482104f1df7.png)

ALB 엔드포인트로 웹에 접속해 *Start Stress* 를 눌러 부하를 주고 CloudWatch에서 ALB의 CPU 사용량을 모니터링 한다.

![image](https://user-images.githubusercontent.com/60495897/140093131-ac5c956c-523e-4849-919f-f0f9a61a2656.png)

다음과 같이 CPU 사용량이 급속도로 증가하고, 최대 용량 만큼 인스턴스 개수가 증가하게 된다. 



> [CloudFormation Code](https://github.com/yuntreee/CloudFormation/blob/46ea4ec6635f5ee39b78c7d6359c6e0c8f6fd0f5/ALB_AutoScaling)



## 2. 단계적 조정 정책

CloudWatch 경보를 사용해 인스턴스의 개수를 단계적으로 조정한다. 설정한 지표의 모니터링된 값에 따라 CloudWatch 경보가 울려 단계 조정 정책이 실행된다. CloudWatch는 기본적으로 5분에 한번 모니터링하며, 최대 1분 간격으로 모니터링할 수 있다

예를 들어, CPU 평균 사용률이 60%가 넘어가면 인스턴스의 개수를 하나 늘리고, CPU 평균 사용률이 60% 이하라면 인스턴스의 개수를 하나 줄인다.



[CloudFormation Code](https://github.com/yuntreee/CloudFormation/blob/ee8b5c8ed2cc4dc0918a22f47f5ed9ca00eff2b8/ALB_AutoScaling(2))