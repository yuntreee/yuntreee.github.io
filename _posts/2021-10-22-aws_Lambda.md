---
title: "[AWS] Lambda"
date: '2021-10-22 22:30:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. AWS Lambda 란?

AWS에서 서버리스 컴퓨팅을 구현하기 위해 **AWS Lambda** 라는 함수를 사용한다. 

말 그대로 호출될 때 원하는 동작을 하도록 작성된 함수이다. 함수가 호출되면 컨테이너 혹은 VM이 생성되어 정의한 함수가 런타임 내에서 실행되고, 결과를 리턴한 후 종료된다.

함수는 서버가 계속 대기하면서 사용자의 요청을 처리하는 것이 아니라, 이벤트가 있을 때마다 실행된다. 리소스를 필요할 때에만 자동으로 사용하고 종료시키기 때문에 **비용 절감의 효과**가 있다.

하지만 AWS Lambda는 **상태유지가 되지 않으며(Stateless)**, 함수가 실행될 때 약간의 **지연시간**이 생긴다. 

Python, Java, Node.js, C#, Go 등의 언어로 AWS Lambda를 작성할 수 있다.



## 2. Life Cycle

<img src="https://user-images.githubusercontent.com/60495897/138304483-a39ee52d-e2cd-4000-bbaa-fca4a021a146.png" alt="image" style="zoom:80%;" />

컨테이너 혹은 Micro VM을 만드는 것을 *Cold Start* 라고 한다. 

이후 환경변수 등에 실행환경을 맞추고, 지정한 언어별 런타임 환경을 준비한다. 이후 작성한 함수를 실행한다.

함수를 실행하면 컨테이너는 잠시 대기상태가 되는데, 이 때 다시 함수를 실행하는 것을 *Warm Start* 라고 한다.

따라서 함수를 주기적으로 호출하도록 스케쥴링한다면 Warm 상태를 유지하여 보다 빠르게 함수가 실행될 수 있다.





## 3. AWS Lambda 사용 예시

<img src="https://user-images.githubusercontent.com/60495897/138307927-d1a8cf22-1f54-45d6-b450-3aa90ce14291.png" alt="image" style="zoom:80%;" />

**조절** : Lambda 실행에 필요한 예약된 동시성을 0으로  초기화한다.

**Layeres** : 추가 코드와 라이브러리를 등록하여 사용한다.

**트리거** : 함수를 실행시킬 서비스나 이벤트이다.

**대상** : 코드를 작성하지 않고도 함수 실행 결과를 전달할 리소스이다.



### 1. S3에서 사진 크기 변환

S3 버킷에 이미지를 넣을 시 Lambda 함수를 통해 사진 크기를 조정한다.



1. Lambda를 실행하고 S3에 접근할 수 있는 역할을 생성한다.

   ![image](https://user-images.githubusercontent.com/60495897/138309590-983fb071-4747-4f9c-9799-11fb7ece85b9.png)

   *AWSLambdaBasicExecutionRole* 과 *AmazonS3FullAccess* 정책을 적용한다.

   

2. S3 버킷에서 변환된 이미지를 가져오기 위한 사용자를 생성한다.

   ![image](https://user-images.githubusercontent.com/60495897/138309935-aac9abb2-eeb1-4561-b682-f4c72e6e6dd2.png)

   *AmazonS3FullAccess* 정책을 부여한다.

   

3. Raw Image를 저장하는 Resized Image를 저장할 2개의 S3를 생성한다.



4. Python을 이용한 Lambda를 생성한다. 이미지를 Resize하여 S3버킷에 업로드하는 코드는 다음과 같다.

   ```python
   import boto3
   import os
   import sys
   import uuid
   from urllib.parse import unquote_plus
   from PIL import Image
   import PIL.Image
   
   s3_client = boto3.client('s3')
   
   def resize_image(image_path, resized_path):
       with Image.open(image_path) as image:
           image.thumbnail(tuple(x / 2 for x in image.size))
           image.save(resized_path)
   
   def lambda_handler(event, context):
       for record in event['Records']:
           bucket = record['s3']['bucket']['name']
           key = unquote_plus(record['s3']['object']['key'])
           tmpkey = key.replace('/', '')
           download_path = '/tmp/{}{}'.format(uuid.uuid4(), tmpkey)
           upload_path = '/tmp/resized-{}'.format(tmpkey)
           s3_client.download_file(bucket, key, download_path)
           resize_image(download_path, upload_path)
           s3_client.upload_file(upload_path, '{}-resized'.format(bucket), key)
   ```



5. PLI와 boto3 라이브러리를 레이어에 추가한다. EC2에서 pip로 라이브러리를 다운받고 S3에 업로드하는 방식을 사용한다.

   ```bash
   # 기존 설치된 PIL 삭제하고 수동 설치된 Python3.7에 다시 설치
   $ rm -rf /usr/lib64/python2.7/site-packages/PIL
   $ yum install -y gcc bzip2-devel ncurses-devel gdbm-devel xz-devel sqlite-devel openssl-devel tk-devel uuid-devel readline-devel zlib-devel libffi-devel
   $ wget https://www.python.org/ftp/python/3.7.0/Python-3.7.0.tar.xz
   $ tar -Jxf Python-3.7.0.tar.xz
   $ cd Python-3.7.0
   $ ./configure --enable-optimizations
   $ make altinstall
   $ export PATH=$PATH:/usr/local/bin
   $ pip3.7 install --upgrade pip
   $ pip3.7 install pillow boto3
   $ ls /usr/local/lib/python3.7/site-packages
   $ mkdir -p /home/ec2-user/lambda_layers/python/lib/python3.7/site-packages
   $ cd /usr/local/lib/python3.7/
   $ cp -r site-packages /home/ec2-user/lambda_layers/python/lib/python3.7/
   $ cd /home/ec2-user/lambda_layers
   $ zip -r lambda_layers.zip *
   
   $ aws configure
   AWS Access Key ID [None]: <사용자 Access Key ID>
   AWS Secret Access Key [None]: <비밀 Access Key>
   $ aws s3 cp lambda_layers.zip <원본버킷>
   ```

   콘솔에서 업로드한 객체의 *객체 URL* 을 사용해 레이어를 생성하고 추가한다.

   <img src="https://user-images.githubusercontent.com/60495897/138315203-87efde0b-b1d9-4b88-ab5d-06e9ec8ab3da.png" alt="image"  />



6. Raw Image용 S3 버킷을 트리거에 추가한다.

   

7. 구성 -> 권한 탭에서 생성한 역할을 선택한다.



8. 함수를 배치(Deploy) 한다.



원본 이미지 버킷에 이미지 파일을 업로드하면 사이즈가 작게 변환된 이미지가 다른 버킷에 저장된 것을 확인할 수 있다.





### 2. Lambda로 정적 웹페이지 호출

Lambda는 API 게이트웨이를 트리거에 연결해서 사용하는 경우가 많다. Lambda로 간단한 정적 웹페이지를 호스팅해본다.



1. Node.js를 사용하는 함수를 생성한다



2. API Gateway 콘솔로 이동하여 REST API를 생성한다. 

   루트에 GET 메서드를 생성하고 Lambda 함수와 연결한다.

   메서드 응답은 다음과 같이 설정한다.

   <img src="https://user-images.githubusercontent.com/60495897/138458251-a1d6bf7a-9e2e-4a44-9721-ef86265041e8.png" alt="image" style="zoom:80%;" />

   통합 응답은 다음과 같이 설정한다.

   <img src="https://user-images.githubusercontent.com/60495897/138459633-c0a99e89-fa02-49fd-9de2-ed7f4bb32fc6.png" alt="image" style="zoom:80%;" />



3. 설정을 완료했다면 API를 배포한다.



<img src="https://user-images.githubusercontent.com/60495897/138460023-ff6a7d09-a13d-4b70-a0af-865d7b4b3c15.png" alt="image" style="zoom:80%;" />

클라이언트가 GET 메서드로 루트 웹페이지를 요청하면 람다함수가 실행되어 위와 같은 정적 웹페이지를 받게 된다.