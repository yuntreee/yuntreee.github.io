---
title: "[AWS] CloudWatch와 Slack 연동"
date: '2022-02-20 15:30:30'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. CloudWatch Alarm과 Slack 연동

SNS와 Lambda를 사용해 CloudWatch 경보를 Slack에서 받아본다.

흐름은 다음 그림과 같다.

![image](https://user-images.githubusercontent.com/60495897/154829883-c0073ec8-1074-43fe-8788-253f7d2f1d86.png){: width="90%" height="90%" .align-center}

CloudWatch Alarm 이 울리면 SNS가 호출되고 이어서 Lambda가 호출된다. Lambda에는 알람 이름과 state에 따라 Slack에 메세지를 생성하여 보내도록 작성된 코드가 있다.

### 1) Slack Webhook 생성

*Slack -> 워크스페이스 메뉴 -> 설정 및 관리 -> 앱 관리*  로 이동한다.

![image](https://user-images.githubusercontent.com/60495897/154829748-fc38316a-2406-49ec-ab9b-6e1219a547c4.png){: width="90%" height="90%" .align-center}



수신 웹후크를 검색해 Slack에 추가한다.

![image](https://user-images.githubusercontent.com/60495897/154829807-40cd83f7-fad2-4e97-b1ee-de8540297612.png){: width="90%" height="90%" .align-center}



추가하면 웹후크용 URL이 생성되는데, 이를 복사해서 저장해둔다.



### 2) SNS 생성

표준 SNS를 생성한다. 설정값은 모두 기본으로 둔다.

![image](https://user-images.githubusercontent.com/60495897/154830210-f9649486-f43e-4b7a-bd26-734bd4184d25.png){: width="90%" height="90%" .align-center}





### 3) Lambda 생성

SNS를 구독할 Lambda를 생성한다. 런타임은 *Node.js 14.X* 를 선택한다.

Slack으로 메세지를 보내는 코드는 아래를 참고한다. 위에서 생성한 Slack Webhook URL을 사용한다.

<script src="https://gist.github.com/yuntreee/92a8c530e2d5357617c3e400b1e4807d.js"></script>



생성 후 SNS 트리거를 추가한다.

![image](https://user-images.githubusercontent.com/60495897/154395556-f68750f9-0e62-40e7-9ab1-efefce1a3067.png){: width="90%" height="90%" .align-center}



트리거 생성 후 SNS 주제를 확인하면 Lambda를 구독중임을 확인할 수 있다.

![image](https://user-images.githubusercontent.com/60495897/154395613-16871894-2fb5-4c28-9f25-346fcfc2172a.png){: width="90%" height="90%" .align-center}



### 4) CloudWatch 경보 생성

CloudWatch 경보를 생성한다. 테스트를 위해 지표는 CPU 사용량이 10% 이상일 때 울리도록 하였다.

![image-20220220150002303](https://user-images.githubusercontent.com/60495897/154833879-2bb21edb-13a0-41fa-899e-7bfab6ec90b4.png)
{: width="80%" height="80%" .align-center}



경보가 울리면 위에서 생성해둔 SNS에 In alarm 상태를 전송한다.

![image-20220220150107857](https://user-images.githubusercontent.com/60495897/154833920-3607bd0a-b379-45cc-a6b3-429605dc978c.png){: width="90%" height="90%" .align-center}



경보 이름은 CPU, MEM, DISK, EBS, NETWORK 중 반드시 하나의 단어만 포함하여야 javascript 함수가 정상적으로  (대소문자 구분 X) 동작한다.



## 2. 결과 확인

정상적으로 동작하는지 확인하기 위해 인스턴스에 접속하여 cpu 부하를 준다.

**stress** 툴을 사용하였다.

```bash
# Amazon-Linxu 2
$ sudo amazon-linux-extras -y epel
$ sudo yum -y install stress

$ stress -c <cpu코어개수>
```



부하를 주고 기다리면 Slack으로 경보 메세지가 온 것을 확인할 수 있다.

![image](https://user-images.githubusercontent.com/60495897/154204053-710eeca6-12be-4ef3-b4c6-c207bbcd0b65.png){: width="90%" height="90%" .align-center}

