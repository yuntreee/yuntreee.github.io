---
title: "[AWS] Cloud9 & CodeCommit"
date: '2021-10-25 23:50:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. AWS Cloud9 이란?

Cloud9은 AWS 내에서 사용하는 IDE 이다. Cloud9 환경을 생성하면 EC2 인스턴스가 생성되고, 이 인스턴스 내에서 Cloud9의 GUI를 활용해 코드 작성을 할 수 있다. UI는 Visual Studio Code와 매우 흡사하다. 

JavaScript, Python, PHP, C/C++, Ruby, Go 등의 다양한 언어를 지원하며, Cloud9환경 생성시 필수 도구가 사전에 패키징되어 제공된다.

서버리스 어플리케이션을 개발할 수 있는 원활한 환경을 제공하므로 손쉽게 서버리스 어플리케이션의 리소스를 정의/디버깅하고, 로컬 실행과 원격 실행 간의 전환을 할 수 있다. 또한 서버리스 개발에 필요한 모든 SDK, 라이브러리 및 플러그인으로 개발 환경을 구성할 수 있다.



## 2. AWS CodeCommit 이란?

CodeCommit은 AWS에서 제공하는 Git 기반 레포지토리이다. 기본적인 사용 방법은 Github과 매우 흡사하다. 

IAM, CloudTrail, CloudWatch와 연동을 통해 데이터에 접근한 유저, 접근 방법, 시간 및 위치까지 제어하고 모니터링 할 수 있다.

HTTP 혹은 SSH를 사용해 파일을 송수신하며, 레포지토리는 KMS를 통해 암호화된다.



## 3. Cloud9 & CodeCommit 연동 구성

### 1) CodeCommit에 파일 업로드

Cloud9 환경에서 작성한 코드를 CodeCommit에 저장한다.



1. Cloud9 환경을 생성한다. 가용영역은 a이다.



2. CodeCommit 레포지토리를 생성한다.



3. EC2에서 AWS CLI를 통해 CodeCommit에 접근하기 위해 IAM 사용자를 생성한다. *CodeCommitFullAccess* 정책을 부여한다.

   또한, HTTPS를 이용해 CodeCommit에 접근할 수 있도록 *보아나 자격 증명*  탭에서 자격 증명을 생성한다.



4. Cloud9에서 AWS CLI 사용자를 생성한 IAM 사용자로 설정한다. 

   IAM 사용자가 HTTPS Git 자격 증명을 사용할 수 있도록 설정한다. 

   ```bash
   $ sudo su
   $ aws configure
     AWS Access Key ID [None]:
     AWS Secret Access Key [None]:
     Default region name [None]: 
     Default output format [None]:
   
   $ git config --global credential.helper '!aws codecommit credential-helper $@'
   $ git config --global credential.UseHttpPath true
   # 해당 내용은 ~/.gitconfig 에 저장된다.
   
   $ exit
   ```



5. CodeCommit 레포지토리를 클론하고, 파일을 하나 작성하여 commit 후 push 한다.

   ```bash
   $ git clone <Repository URL>
   $ git add .
   $ git commit -m "Comment"
   $ git push
   ```

   레포지토리에 작성한 파일이 업로드되어 있음을 볼 수 있다.





### 2) Github 저장소를 CodeCommit 저장소로 마이그레이션

1. CodeCommit 레포지토리를 생성한다.



2. ```~/environment```에 디렉토리를 하나 생성 후 Github 레포지토리를 *mirror* 옵션을 주어 클론한다.

```bash
$ git clone --mirror <Github 저장소 URL>
```



3. 클론한 데이터를 CodeCommit으로 push 한다.

```bash
$ git push --all <CodeCommit 저장소 URL>
```



Github 저장소의 파일이 CodeCommit에 동일하게 있음을 확인한다.

