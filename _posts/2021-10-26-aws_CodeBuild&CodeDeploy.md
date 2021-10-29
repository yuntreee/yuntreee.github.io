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



## 4. CodeDeploy를 사용한 웹 배포



1. CodeDeploy 역할을 하나 생성한다. 다음은 모든 리전에서 CodeDeploy에 접근 가능하게 하는 신뢰관계 JSON 이다.

   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Sid": "",
               "Effect": "Allow",
               "Principal": {
                   "Service": [
                       "codedeploy.us-east-2.amazonaws.com",
                       "codedeploy.us-east-1.amazonaws.com",
                       "codedeploy.us-west-1.amazonaws.com",
                       "codedeploy.us-west-2.amazonaws.com",
                       "codedeploy.eu-west-3.amazonaws.com",
                       "codedeploy.ca-central-1.amazonaws.com",
                       "codedeploy.eu-west-1.amazonaws.com",
                       "codedeploy.eu-west-2.amazonaws.com",
                       "codedeploy.eu-central-1.amazonaws.com",
                       "codedeploy.ap-east-1.amazonaws.com",
                       "codedeploy.ap-northeast-1.amazonaws.com",
                       "codedeploy.ap-northeast-2.amazonaws.com",
                       "codedeploy.ap-southeast-1.amazonaws.com",
                       "codedeploy.ap-southeast-2.amazonaws.com",
                       "codedeploy.ap-south-1.amazonaws.com",
                       "codedeploy.sa-east-1.amazonaws.com"
                   ]
               },
               "Action": "sts:AssumeRole"
           }
       ]
   }
   ```

    

2. EC2가 S3에 접근할 수 있는 AmazonS3FullAccess 정책을 부여한 역할을 생성한다. 



3. EC2 Auto Scailing을 구성한다. (2)에서 생성한 역할을 *IAM 인스턴스 프로파일* 로 지정한다.

   보안그룹은 HTTP/HTTPS 를 허용하고, 다음과 같은 사용자 데이터로 CodeDeploy가 설치되게 설정한다.

   ```bash
   #!/bin/bash
   yum -y update
   yum install -y ruby
   cd /home/ec2-user
   curl -O https://aws-codedeploy-ap-northeast-2.s3.amazonaws.com/latest/install
   chmod +x ./install
   sudo ./install auto
   ```



​		설정한 시작구성을 사용하여 가용영역 A,C에 Auto Scailing 그룹을 생성한다.



4. 가용영역 A에 Cloud9 환경을 생성한다. 폴더를 생성하고 간단한 *index.html*을 작성한다. 



5. CodeDeploy 배포를 위해서는 json 혹은 yaml 형식의***appspec*** 파일이 필요하다. 

   배포시 각 단계마다 실행할 명령 정보를 작성한다고 생각하면 된다.

   ```yaml
   # appspec.yml
   version: 0.0
   os: linux
   files:
     - source: /index.html
       destination: /var/www/html/
   hooks:
     BeforeInstall:
       - location: scripts/install_dependencies.sh
         timeout: 300
         runas: root
       - location: scripts/start_server.sh
         timeout: 300
         runas: root
     ApplicationStop:
       - location: scripts/stop_server.sh
         timeout: 300
         runas: root
   ```



6. *appspec.yml*에 정의된 파일들을 생성한다.

   ```bash
   # scripts/install_dependencies.sh
   #!/bin/bash
   yum install -y httpd
   
   # scripts/start_server.sh
   #!/bin/bash
   service httpd start
   
   # scripts/stop_server.sh
   #!/bin/bash
   isExistApp = `pgrep httpd`
   if [[ -n  $isExistApp ]]; then
       service httpd stop        
   fi
   ```

<img src= 'https://user-images.githubusercontent.com/60495897/139246891-087ad53c-29a2-40c2-945b-96d0bbcddf19.png' align=center>

​		

7. 위 디렉토리를 .zip 파일로 압축 후 S3에 업로드한다.

   ```bash
   $ zip -r codedeploy.zip *
   $ aws s3 cp codedeploy.zip s3://버킷명
   ```

   

8. CodeDeploy EC2어플리케이션과 배포그룹을 생성한다. 

   배포그룹에서 *블루/그린* 은 failover를 위해 사용된다. (블루=master, 그린=slave)

   Auto Scailing 그룹을 선택한다.



9. S3의 *.zip* 파일을 배포하도록 배포생성한다. 



배포가 성공한다면 성공적으로 웹 화면을 볼 수 있다. 

만약 변경 내용이 있다면 다시 압축하며 S3로 전송한 후 재배포한다.



현재 배포 대상이 Auto Scailing 그룹으로 되어있다. Auto Scailing 그룹의 인스턴스를 추가적으로 증가 후 재배포하면 추가된 인스턴스에도 같은 환경으로 어플리케이션이 배포됨을 확인할 수 있다.