---
title: "[AWS] API Gateway - api key 사용하기"
date: "2022-12-20 15:40:30"
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---

## API Key란

API Gateway에 key를 등록하여, 해당 key를 요청에 포함할 때에만 api 호출을 허용할 수 있도록 보안을 강화할 수 있다.

또한, key 별로 api 호출 수 제한을 설정해 과도한 api 호출을 제한할 수도 있다.

### 1) API 응답 Lambda 생성

**{"message" : "hello"}** 를 반환하는 간단한 Python 람다를 생성한다.

```python
import base64
import json
import logging
import os

logger = logging.getLogger()
handler = logging.StreamHandler()
formatter = logging.Formatter("[%(levelname)s] %(message)s")
handler.setFormatter(formatter)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info(event)
    return {
        'statusCode': 200,
        'headers': {
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Headers': 'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token',
            'Access-Control-Allow-Credentials': 'true',
            'Content-Type': 'application/json'
        },
        'body': json.dumps({"message" : "hello"})
    }
```



### 2) API Gateway 생성

API Gateway -> REST API를 하나 생성한다.

![image](https://user-images.githubusercontent.com/60495897/208563476-12872433-c925-41ce-99a7-ed395577a5bf.png){: width="90%" height="90%" .align-center}

![image](https://user-images.githubusercontent.com/60495897/208563577-3e315b96-0310-44ae-9585-42d27dcba26f.png){: width="90%" height="90%" .align-center}


**[1]** 에서 생성한 Lambda를 호출하는 GET 메소드를 구성한다.

![image](https://user-images.githubusercontent.com/60495897/208564824-56528912-525b-4679-8923-13cc13eb58d0.png){: width="90%" height="90%" .align-center}


메소드를 요청할 때 API Key를 요구하도록 설정한다.

![image](https://user-images.githubusercontent.com/60495897/208565415-fe57e314-7ee8-4134-9357-3884b446d15f.png){: width="90%" height="90%" .align-center}

![image](https://user-images.githubusercontent.com/60495897/208565544-159211fa-682b-427a-a96a-0ee7f715fd09.png){: width="90%" height="90%" .align-center}


설정 완료된 API를 배포한다.

![image](https://user-images.githubusercontent.com/60495897/208564941-6192fb39-b6c9-4836-a745-56da2d883fee.png){: width="90%" height="90%" .align-center}


### 3) API Key, Usage Plan 구성

**API Keys** 메뉴에서 새로운 API key를 하나 생성한다.

![image](https://user-images.githubusercontent.com/60495897/208563903-6e5e8c8e-a4e6-4d82-8084-13a3c3c5cce1.png){: width="90%" height="90%" .align-center}


Key를 API 리소스에 연결하기 위해서는 **Usage Plan** 을 구성해야한다.

**Usage Plan** 은 api 호출 빈도나 호출 수를 제한하기 위해 사용된다.

이 글에서는 key를 사용해 API Gateway를 호출할 경우에 대해 제한해본다.

![image](https://user-images.githubusercontent.com/60495897/208564243-f7a60e75-88a8-4529-a809-44b655209bc3.png){: width="90%" height="90%" .align-center}

**Throttling** 은 초당 호출 수를 제한하고,

**Quota** 는 일/월/년 동안 최대 호출 수를 제한한다.


![image](https://user-images.githubusercontent.com/60495897/208565096-206c1850-3470-4b45-a4a5-291cf52e3581.png){: width="90%" height="90%" .align-center}

Usage Plan이 적용될 API와 스테이지를 추가하는 화면이다.


![image](https://user-images.githubusercontent.com/60495897/208565234-0a369a53-c636-4616-b5fe-953e9ab00d31.png){: width="90%" height="90%" .align-center}

Usage Plan이 인증에 사용할 수 있는 API Key를 추가하는 화면이다.


_만약 Usage Plan에 여러 key를 등록한다면, Throttling/Quota 제한은 각 key별로 적용된다._


### 4) 테스트

![image](https://user-images.githubusercontent.com/60495897/208581107-4bed5910-51d1-4c91-83ad-e7497d9c42b7.png){: width="90%" height="90%" .align-center}

Stages 메뉴에 들어가면 배포한 API의 URL 이 있다.


![image](https://user-images.githubusercontent.com/60495897/208581276-7b940608-ae8d-4714-bd0f-2664e5837d86.png){: width="90%" height="90%" .align-center}

PostMan에서 별다른 인증 없이 api를 호출하면 접근이 제한된다.


![image](https://user-images.githubusercontent.com/60495897/208581372-e0dd1aa0-a2a2-4054-9c18-9befca8ba69d.png){: width="90%" height="90%" .align-center}

x-api-key 에 **[3]** 에서 생성한 키값을 넣으면 정상 동작함을 확인할 수 있다.
