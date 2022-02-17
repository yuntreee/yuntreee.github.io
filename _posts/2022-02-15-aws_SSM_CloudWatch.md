---
title: "[AWS] SSM으로 CloudWatch Agent 설치/실행"
date: '2022-02-15 22:06:30'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. System Manager를 사용하여 CloudWatch Agent 설치

AWS System Manager(SSM)는 관리형 노드들을 제어하기 위해 사용하는 서비스이다. EC2 이외에도 온프레미스나 타 클라우드 환경의 VM을 지원한다.

이 글에서는 SSM을 사용하여 인스턴스에 CloudWatch Agent를 설치하고 모니터링 해본다.



### 1) EC2 역할 생성

인스턴스가 CloudWatch Agent와 SSM을 사용할 수 있도록 역할을 부여한다. 

**CloudWatchAgentServerPolicy** 와 **AmazonSSMManagedInstanceCore** 정책을 사용한다.

![image](https://user-images.githubusercontent.com/60495897/153786251-8133f458-465c-43a4-8386-93f216de954b.png){: width="90%" height="90%" .align-center}



### 2) SSM Agent 구동 확인

Amazon Linux 2에는 SSM Agent가 기본적으로 설치된다. 

```bash
[root@backend ec2-user]# systemctl status amazon-ssm-agent
● amazon-ssm-agent.service - amazon-ssm-agent
   Loaded: loaded (/usr/lib/systemd/system/amazon-ssm-agent.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2022-02-14 10:36:07 KST; 13min ago
 Main PID: 2227 (amazon-ssm-agen)
   CGroup: /system.slice/amazon-ssm-agent.service
           ├─2227 /usr/bin/amazon-ssm-agent
           └─2393 /usr/bin/ssm-agent-worker

```

다른 운영체제라면 다음 링크를 참조하여 SSM Agent를 수동설치한다.

https://docs.aws.amazon.com/ko_kr/systems-manager/latest/userguide/sysman-manual-agent-install.html





### 3) SSM 사용하여 CloudWatch Agent 다운로드

*System Manager -> 노드관리->명령실행 -> 명령 실행*  으로 이동한다.

![image](https://user-images.githubusercontent.com/60495897/153787463-a8a7dc4b-18ce-4616-82bc-af6d463608fe.png){: width="90%" height="90%" .align-center}



명령 문서는 **AWS-ConfigureAWSPackage** 을 선택한다.

![image](https://user-images.githubusercontent.com/60495897/153788378-eed4a1a0-2c7e-47aa-9559-29b792272ecb.png){: width="90%" height="90%" .align-center}



명령 파라미터는 다음과 같다.

Action : Install

Installation Type : Uninstall and resinstall

Name : AmazonCloudWatchAgent

Version: latest

![image](https://user-images.githubusercontent.com/60495897/153789675-8cf7fe85-a45c-4f27-8725-cc8eb929a9e3.png){: width="90%" height="90%" .align-center}



해당 명령을 실행할 대상을 선택한다. 인스턴스 태그를 통해 지정하거나 수동으로 선택할 수도 있다.

![image](https://user-images.githubusercontent.com/60495897/154063221-067d3bfc-e49a-4945-a390-9e518b9a7be6.png){: width="90%" height="90%" .align-center}

![image](https://user-images.githubusercontent.com/60495897/154063528-d3eb7841-eab4-4292-8d20-508d6d916454.png){: width="90%" height="90%" .align-center}



명령을 실행하고, CloudWatch Agent가 설치되었는지 확인한다.

![image](https://user-images.githubusercontent.com/60495897/153973854-a9457e28-85f0-4a12-a83f-71245287e2e3.png){: width="90%" height="90%" .align-center}

```bash
[root@ip-10-0-0-10 ec2-user]# systemctl status amazon-cloudwatch-agent.service
● amazon-cloudwatch-agent.service - Amazon CloudWatch Agent
   Loaded: loaded (/etc/systemd/system/amazon-cloudwatch-agent.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
```



 현재 인스턴스에 CloudWatch Agent가 설치는 됐지만 실행은 안된 상태이다. 

config.json을 통해 원하는 지표를 CloudWatch로 보내도록 설정 후 실행하는 과정이 필요하다.



### 4) Parameter Store에 config.json 저장

*System Manager -> 애플리케이션 관리 -> Parameter Store -> 파라미터 생성* 으로 이동한다.

파라미터 이름과 CloudWatch config.json 파일 내용을 입력한다.

```bash
#config.json 예시
{
    "agent": {
        "metrics_collection_interval": 60,
        "run_as_user": "cwagent"
    },
    "metrics": {
        "append_dimensions": {
            "InstanceId": "${aws:InstanceId}"
        },
        "metrics_collected": {
            "disk": {
                "measurement": [
                    {
                        "name": "disk_used_percent",
                        "rename": "DiskSpaceUtilization",
                        "unit": "Percent"
                    }   
                ],
                "metrics_collection_interval": 60,
                "resources": [
                    "*"
                ]
            },
            "mem": {
                "measurement": [
                    {
                        "name": "mem_used_percent",
                        "rename": "MemoryUtilization",
                        "unit": "Percent"
                    }
                ],
                "metrics_collection_interval": 60
            }
        },
        "namespace": "CWAgent/Linux"
    }
}
```

![image](https://user-images.githubusercontent.com/60495897/154065264-6d067c22-f130-4a6d-bdfb-19b8d7983856.png){: width="90%" height="90%" .align-center}



### 5) SSM을 사용하여 CloudWatch 실행

*System Manager -> 노드관리 -> 명령 실행 -> 명령 실행* 으로 이동한다.

명령 문서는 **AmazonCloudWatch-ManageAgent** 를 선택한다.

![image](https://user-images.githubusercontent.com/60495897/153802268-de7e5f05-7c02-4bff-8967-53ac545ab25e.png){: width="90%" height="90%" .align-center}



명령 파라미터는 다음과 같다.

Action : configure

Mode : ec2

Optional Configuration Source : ssm

Optional Configuration Location : 파라미터 스토어에 저장한 파라미터 이름

Optional Open Telemetry Collector Configuration Source : ssm

Optional Restart : yes

![image](https://user-images.githubusercontent.com/60495897/154066347-ce90ceaf-dc1d-4b91-b4bf-e06fab75fdf9.png){: width="90%" height="90%" .align-center}



인스턴스를 지정하고 명령을 실행한다.



![image](https://user-images.githubusercontent.com/60495897/153974950-66bdabd7-1abf-4585-8fce-22192d65adb2.png){: width="90%" height="90%" .align-center}





## 2. 결과 확인

CloudWatch -> 지표 -> 모든 지표 로 이동하여 config.json에서 설정한 Custom Namespace 가 생성되었는지 확인한다.

![image](https://user-images.githubusercontent.com/60495897/154067456-16f3c369-cd2f-4d03-87cb-9c9b77516460.png){: width="90%" height="90%" .align-center}