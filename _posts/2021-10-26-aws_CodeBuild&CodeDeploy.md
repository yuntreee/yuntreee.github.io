---
title: "[AWS] CodeBuild & CodeDeploy"
date: '2021-10-26 01:19:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. CodeBuild 란?

개발 코드를 컴파일하는 단계에서부터 테스트 후 배포까지의 단계를 지원하는 CI (Continuous Integration) 서비스이다. 자체 빌드 서버가 필요하지 않으며, 빌드 서버를 프로비저닝하거나 운영/관리 및 확장을 수행할 필요가 없다.



## 2. CodeDeploy 란?

EC2, ECS, Lambda 및 온프레미스 서버에 대해 소프트웨어 배포를 자동화하는 CD (Continuous Delivery) 서비스이다. 배포그룹을 생성하여 동일한 소프트웨어를 배포한다.



## 3. CodeBuild를 사용한 S3 정적 웹 호스팅

1. Cloud9과 CodeCommit 레포지토리를 생성한다.



2. 정적 웹사이트 호스팅을 위해 *vue*와 *vue-cli* 를 설치한다.

   ```bash
   # npm은 js패키지 인스톨러
   $ sudo npm install vue
   $ sudo npm install --global vue-cli
   ```



3. CodeCommit 레포지토리를 clone하고 vue webpack을 다운받는다.

   ```bash
   $ git clone <CodeCommit URL>
   $ vue init webpack <클론한 디렉토리>
     설정값은 모두 Y
   $ git add .
   $ git commit -m "Comment"
   $ git push
   ```



4. 퍼블릭 접근이 가능한 S3 버킷을 생성한다. 

   *속성*  탭에서 *정적 웹사이트 호스팅*을 활성화한다.  인덱스 문서는 *index.html* 로 설정한다.

   *권한* 탭에서 다음과 같은 정책을 추가한다.

```json
# 버킷에 대한 사용자들의 Read를 허용한다.
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::Bucket-Name/*"
            ]
        }
    ]
}
```



5. CodeBuild를 생성한다.

   생성한 레포지토리를 선택하고 브랜치는 master이다.

   *운영체제* 는 *Amazon Linux 2*, *런타임* 은 *Standard*, *이미지* 는 *2.0* 버전을 선택한다.

   *아티팩트* 는 생성해놓은 S3 버킷을 선택한다.



6. CodeBuild를 생성하면 IAM 역할이 하나 생성된다. 

   CodeBuild가 S3에 접근할 수 있도록 *AmazonS3FullAccess* 정책을 부여한다.



7. 클론한 디렉토리에 빌드 정보가 담긴 *buildspec.yml* 파일을 작성하고 push 한다.

   ```bash
   version: 0.2
   
   phases:
     install:
       runtime-versions:
         nodejs: 10
       commands:
         - npm i npm@latest -g
     pre_build:
       commands:
         - npm install
     build:
       commands:
         - npm run build
     post_build:
       commands:
         - aws s3 sync ./dist s3://Bucket-Name
   ```



8. 빌드 후 S3 정적 웹사이트 호스팅 엔드포인트로 접속하여 확인한다.

   <img src="https://user-images.githubusercontent.com/60495897/138732196-532af77e-a9da-43bd-97ad-68af71daf0ab.png" alt="image"  />

