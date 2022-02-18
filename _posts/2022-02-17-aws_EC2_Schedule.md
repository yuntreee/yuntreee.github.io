---
title: "[AWS] Lambda로 EC2 시작/종료 자동화"
date: '2022-02-16 21:43:30'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. Lambda로 EC2 시작/종료 자동화

Lambda와 EventBridge를 사용해 EC2 인스턴스 시작/종료를 자동화한다.



lambda에 부여할 역할을 생성한다.

적용되는 정책은 다음과 같다. 

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Start*",
        "ec2:Stop*",
        "ec2:Describe*"
      ],
      "Resource": "*"
    }
  ]
}
```



### 1) Start 함수

생성한 Lambda 역할을 사용해 Python 3.9 를 생성한다. 함수 Timeout은 넉넉하게 30초로 변경한다.

함수는 다음과 같다.

```python
# auto-schedule : True 태그값이 있는 인스턴스에 적용

import boto3
region = 'ap-northeast-2'
instances = []
ec2_r = boto3.resource('ec2')
ec2 = boto3.client('ec2', region_name=region)

for instance in ec2_r.instances.all():
    for tag in instance.tags:
        if tag['Key'] == 'auto-schedule':
            if tag['Value'] == 'True':
                instances.append(instance.id)

def lambda_handler(event, context):
    ec2.start_instances(InstanceIds=instances)
    print('started your instances: ' + str(instances))
```

설정한 리전의 모든 인스턴스들 중 태그가 *auto-schedule : True* 인 인스턴스를 찾아 *ec2.start_instances()* 를 실행하게 된다.

함수 작성 후, 정상적으로 동작하는지 확인하기 위해 Test 를 해본다.



cron 형식 EventBridge로 트리거를 생성한다. 

cron은 ***분 시 일 월 요일 년도*** 형식이다. UTC 기준 시간이며, ***일*** 과 ***요일*** 중 하나는 값이 반드시 **물음표(?)** 여야 한다.

cron 예시는 다음과 같다.

|            빈도             |           표현식           |
| :-------------------------: | :------------------------: |
|         매일 10:15          |     cron(15 10 * * ?)      |
| 월요일부터 금요일까지 18:00 |  cron(0 18 ? * MON-FRI *)  |
|    매월 첫날 오전 08:00     |     cron(0 8 1 * ? *)      |
|        평일 10분마다        | cron(0/10 * ? * MON-FRI *) |
|   매월 첫째 일요일 09:00    |    cron(0 9 ? * 2#1 ?)     |

예를 들어 한국시간 기준 매일 오전 9시에 함수를 실행하는 식은 **cron(0 0 * * ? *)** 이다.





### 2) Stop 함수

과정은 위와 동일하며, 함수 구문과 cron 값만 다르다.

함수는 다음과 같다.

```python
# auto-schedule : True 태그값이 있는 인스턴스에 적용

import boto3
region = 'ap-northeast-2'
instances = []
ec2_r = boto3.resource('ec2')
ec2 = boto3.client('ec2', region_name=region)

for instance in ec2_r.instances.all():
    for tag in instance.tags:
        if tag['Key'] == 'auto-schedule':
            if tag['Value'] == 'True':
                instances.append(instance.id)

def lambda_handler(event, context):
    ec2.stop_instances(InstanceIds=instances)
    print('stopped your instances: ' + str(instances))
```



매일 오후 6시에 함수를 실행하는 식은 **cron(0 9 * * ? *)** 이다.