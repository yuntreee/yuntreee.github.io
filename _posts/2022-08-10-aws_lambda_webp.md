---
title: "[AWS] Lambda@Edge로 이미지 webp 변환"
date: "2022-08-10 00:25:30"
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---

## 1. 개요

Lambda@Edge 는 전 세계에 위치하는 AWS Edge Location에 배포되어 동작하는 Lambda 함수이다.

버지니아 리전에 생성된 Lambda가 Edge Location으로 복제되는 방식이다.

이 글에서는 이미지를 webp 확장자로 변환하여 서비스하는 방법을 소개한다.

![image](https://user-images.githubusercontent.com/60495897/183667091-3ce2535f-af0b-4f14-a164-3964bc340755.png){: width="70%" height="70%" .align-center}

사용자가 CloudFront를 통해 이미지 파일을 요청하면, CloudFront는 Origin S3에게 해당 이미지를 요청한다.

Lambda 함수는 S3의 응답을 받아 webp로 변환 후 사용자에게 응답한다.

변환된 이미지는 CloudFront에 캐시로 남아 동일한 요청이 오면 Lambda 함수가 동작하지 않고 캐시로 응답한다.

## 2. 구성

### 1) S3 버킷 생성

프라이빗 S3 버킷을 생성한다.

![image](https://user-images.githubusercontent.com/60495897/183281664-146c1bd1-7d14-4918-9b3a-e2ed5bbb6562.png){: width="90%" height="90%" .align-center}

테스트 jpg 이미지를 업로드해놓는다.

![image](https://user-images.githubusercontent.com/60495897/183281889-d9f99591-df62-4e57-aa3f-76860387aecf.png){: width="90%" height="90%" .align-center}

### 2) CloudFront 생성

이 S3를 원본으로 하는 CloudFront 배포를 생성한다. 프라이빗 S3에 접근하기 위해 OAI를 설정한다.

![image](https://user-images.githubusercontent.com/60495897/183281979-eac55e44-ceae-4bd7-8ce6-d61877ed67bd.png){: width="90%" height="90%" .align-center}

CloudFront로 요청하면, 지금은 jpg확장자로 응답된다.

![image](https://user-images.githubusercontent.com/60495897/183282554-754f527d-1128-4a69-94c2-543c1ccc9288.png){: width="90%" height="90%" .align-center}

### 3) Lambda 권한 설정

Lambda가 사용할 IAM 역할을 생성한다.

정책은 아래와 같다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateServiceLinkedRole",
        "lambda:GetFunction",
        "lambda:EnableReplication",
        "cloudfront:UpdateDistribution",
        "s3:GetObject",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
```

lambda 뿐 아니라 lambda@edge도 사용할 수 있게 신뢰관계를 아래와 같이 편집한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": ["edgelambda.amazonaws.com", "lambda.amazonaws.com"]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

이 역할을 사용하는 **Node.js 14** Lambda 를 생성한다.

![image](https://user-images.githubusercontent.com/60495897/183681123-37c35efa-0856-4dc3-a879-ff8b2c11b6ff.png){: width="90%" height="90%" .align-center}

Lambda 함수가 프라이빗 S3 버킷에 접근할 수 있어야 한다.

위에서 생성한 Lambda 역할에 대해 접근을 허용하도록 버킷 정책을 편집한다.

```json
{
...
        {
            "Sid": "GetObject",
            "Effect": "Allow",
            "Principal": {
                "AWS": "<Lambda Role ARN>"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<버킷명>/*"
        },
        {
            "Sid": "ListBucket",
            "Effect": "Allow",
            "Principal": {
                "AWS": "<Lambda Role ARN>"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::<버킷명>"
        }
...
}
```

### 4) Lambda 함수 작성

이제 Lambda 함수를 작성해본다.

npm 패키지를 설치해야 하는데, 이 작업은 로컬에서 해도 무방하다.

이 글에서는 Cloud9 환경에서 진행한다.

Lambda를 Cloud9으로 다운로드한다.

![image](https://user-images.githubusercontent.com/60495897/183293845-3af4d4d1-e2e8-4a95-b0ab-4da8c3f0eb03.png){: width="60%" height="60%" .align-center}

아래의 함수를 사용한다.

```javascript
"use strict";

const querystring = require("querystring");
const AWS = require("aws-sdk");
const Sharp = require("sharp");

const S3 = new AWS.S3({
  signatureVersion: "v4",
  region: "ap-northeast-2", // 버킷을 생성한 리전 입력
});

// 버킷명
const BUCKET = "버킷명";

// 호환 가능한 확장자
const supportImageTypes = ["jpg", "jpeg", "png", "gif", "webp", "svg", "tiff"];

exports.handler = async (event, context, callback) => {
  const { request, response } = event.Records[0].cf;

  const { uri } = request;
  const ObjectKey = decodeURIComponent(uri).substring(1);

  const extension = uri.match(/\/?(.*)\.(.*)/)[2].toLowerCase();
  let s3Object;
  let resizedImage;

  // 호환 가능한 확장자가 아니면 원본 반환
  if (!supportImageTypes.some((type) => type === extension)) {
    return callback(null, response);
  }

  // Verify For AWS CloudWatch.
  console.log("S3 Object key:", ObjectKey);

  try {
    s3Object = await S3.getObject({
      Bucket: BUCKET,
      Key: ObjectKey,
    }).promise();

    console.log("S3 Object:", s3Object);
  } catch (error) {
    // 해당 이미지가 없을 경우의 응답
    responseHandler(404, "Not Found", "The image does not exist.", [
      { key: "Content-Type", value: "text/plain" },
    ]);
    return callback(null, response);
  }

  // sharp로 이미지 webp 변환
  try {
    resizedImage = await Sharp(s3Object.Body).webp().toBuffer();
  } catch (error) {
    responseHandler(500, "Internal Server Error", "Fail to resize image.", [
      {
        key: "Content-Type",
        value: "text/plain",
      },
    ]);
    return callback(null, response);
  }

  // 응답 이미지 용량이 1MB 이상일 경우 원본 반환.
  if (Buffer.byteLength(resizedImage, "base64") >= 1048576) {
    return callback(null, response);
  }

  responseHandler(
    200,
    "OK",
    resizedImage.toString("base64"),
    [
      {
        key: "Content-Type",
        value: `image/webp`,
      },
    ],
    "base64"
  );

  /**
   * @summary response 객체 수정을 위한 wrapping 함수
   */
  function responseHandler(
    status,
    statusDescription,
    body,
    contentHeader,
    bodyEncoding
  ) {
    response.status = status;
    response.statusDescription = statusDescription;
    response.body = body;
    response.headers["content-type"] = contentHeader;
    if (bodyEncoding) {
      response.bodyEncoding = bodyEncoding;
    }
  }

  console.log("Success resizing image");

  return callback(null, response);
};
```

sharp 패키지를 설치한다.

```bash
yuntreee:~/environment $ cd convert_webp/
yuntreee:~/environment/convert_webp $ npm init -y
yuntreee:~/environment/convert_webp $ npm install -y sharp
```

작성된 어플리케이션을 Lambda로 업로드한다.

![image](https://user-images.githubusercontent.com/60495897/183294079-4142fb21-f4c9-4f4b-896c-3c1a0d15e9bc.png){: width="60%" height="60%" .align-center}

![image](https://user-images.githubusercontent.com/60495897/183294099-45e8bbcb-3867-4cc4-ad9b-223b53ff77a9.png){: width="60%" height="60%" .align-center}

![image](https://user-images.githubusercontent.com/60495897/183294124-745a90ac-adfd-44b8-8e18-ecacc92f92bb.png){: width="60%" height="60%" .align-center}

![image](https://user-images.githubusercontent.com/60495897/183294150-a937f470-4388-4978-a369-8bc7e3647880.png){: width="60%" height="60%" .align-center}

#### 5) Lambda@Edge 배포

업로드가 완료되면 이제 Lambda@Edge로 배포한다.

![image](https://user-images.githubusercontent.com/60495897/183294182-0ab5eddf-c58b-4215-a8e1-f34f0ba685d1.png){: width="90%" height="90%" .align-center}

CloudFront event는 **Origin Response** 로 설정한다. 원본 (여기서는 S3 버킷)의 응답에 대해 Lambda가 동작한다.

![image](https://user-images.githubusercontent.com/60495897/183294261-caca862f-9acf-4488-87d7-e6ef785df7f0.png){: width="90%" height="90%" .align-center}

_배포한 CloudFront 동작->편집_ 을 확인해보면 Lambda@Edge가 배포되어있다.

![image](https://user-images.githubusercontent.com/60495897/183294299-c07a0e98-ac1a-461f-ae93-7b1f753cf4d3.png){: width="90%" height="90%" .align-center}

![image](https://user-images.githubusercontent.com/60495897/183294318-225affae-9c17-4bb6-8314-554da1541e74.png){: width="90%" height="90%" .align-center}

동일한 이미지로 테스트하면 CloudFront에 남아있는 jpg 캐시가 반환되기 때문에 캐시를 삭제해준다.

![image](https://user-images.githubusercontent.com/60495897/183295298-68246984-2745-47f9-870f-8ff0630d11fa.png){: width="90%" height="90%" .align-center}

![image](https://user-images.githubusercontent.com/60495897/183295320-9d894d92-064a-46d8-84bd-93900518290c.png){: width="90%" height="90%" .align-center}

## 3. 결과 확인

CloudFront로 jpg 파일을 요청하면 webp로 변환된 이미지를 확인할 수 있다.

첫 요청때는 Lambda가 동작하느라 Latency가 조금 있지만, 이후 요청은 CloudFront에 저장된 캐시로 빠르게 응답받는다.

![image](https://user-images.githubusercontent.com/60495897/183298822-c7afc9aa-20c9-4dca-887b-0bfd4a3634e1.png){: width="90%" height="90%" .align-center}
