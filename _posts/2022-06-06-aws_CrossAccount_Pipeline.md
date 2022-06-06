---
title: "[AWS] 교차 계정 CodePipeline 구성"
date: "2022-06-06 16:38:30"
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---

## 1. 교차 계정 파이프라인 구조

개발자들이 PROD 인프라 계정에는 보안상 접근이 제한된 경우가 있다.

이럴 경우 DEV 계정에 있는 파이프라인에서 PROD 계정 쪽으로 배포할 수 있도록 구성할 수 있다.

빌드까지는 DEV 계정에서 진행된 후, cross account role을 사용해 PROD 계정의 CodeDeploy로 접근 후 배포하는 구조이다.

![image](https://user-images.githubusercontent.com/60495897/172119549-6a3d5260-d795-42dd-9f44-3dd601e6dfa9.png){: width="90%" height="90%" .align-center}

위 그림을 참고하면 이해에 도움이 된다.

## 2. 서비스 역할 및 정책 설정

<span style='background-color: #f1f8ff'> **DEV 계정** </span>에 **CodeCommit**과 **CodeBuild**가 있고, <span style='background-color: #ffdce0'> **PROD 계정** </span>에 **CodeDeploy**가 있는 상황에서 이들을 하나의 파이프라인으로 묶는 과정이다.

### [1] KMS 생성

<span style='background-color: #f1f8ff'> **DEV 계정** </span> 에서 고객 관리형 KMS를 생성한다.

![image](https://user-images.githubusercontent.com/60495897/172110146-c1b3b672-29f2-4c67-b1ad-bb9d7f685cd4.png){: width="90%" height="90%" .align-center}

키 설정은 기본값으로 두고 관리자를 선택 후, KMS키를 사용할 사용자(역할)을 선택하는 화면에서 **CodePipeline이 사용하는 역할**과 <span style='background-color: #ffdce0'> **PROD 계정** </span> 을 추가한다.

CodePipeline과 PROD계정은 생성된 고객 관리형 KMS를 통해 Artifact S3에 접근하게 된다.

### [2] Artifact S3 정책 설정

<span style='background-color: #f1f8ff'> **DEV 계정** </span> 에 위치한 Artifact S3에 다음과 같이 접근 정책을 설정한다.

```json
{
  "Version": "2012-10-17",
  "Id": "SSEAndSSLPolicy",
  "Statement": [
    {
      "Sid": "DenyUnEncryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "<코드파이프라인 ARN>/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyInsecureConnections",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "<코드파이프라인 ARN>/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": false
        }
      }
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<PROD 계정 ID>:root"
      },
      "Action": ["s3:Get*", "s3:Put*"],
      "Resource": "<코드파이프라인 ARN>/*"
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<PROD 계정 ID>:root"
      },
      "Action": "s3:ListBucket",
      "Resource": "<코드파이프라인 ARN>"
    }
  ]
}
```

이제 해당 S3에는 CodePipeline과 PROD계정이 KMS를 사용해 접근할 수 있다.

### [3] 교차 계정 IAM 역할 생성

<span style='background-color: #ffdce0'> **PROD 계정** </span>에서 <span style='background-color: #f1f8ff'> **DEV 계정** </span> 이 사용할 수 있는 역할을 생성한다.

![image](https://user-images.githubusercontent.com/60495897/172109651-135cbdd9-c82a-4e03-851c-4bbe385eba9d.png){: width="90%" height="90%" .align-center}

다음과 같은 인라인 정책을 추가한다.

1. **아티팩트 S3에 접근**

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "",
         "Effect": "Allow",
         "Action": [
           "s3:Put*",
           "s3:Get*",
           "codecommit:ListRepositories",
           "codecommit:ListBranches"
         ],
         "Resource": "<Artifact S3 버킷 ARN>/*"
       }
     ]
   }
   ```

2. **KMS 사용**

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "",
         "Effect": "Allow",
         "Action": [
           "kms:ReEncrypt*",
           "kms:GenerateDataKey*",
           "kms:Encrypt",
           "kms:DescribeKey",
           "kms:Decrypt"
         ],
         "Resource": "<KMS ARN>"
       }
     ]
   }
   ```

3. **CodeDeploy 사용**

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "",
         "Effect": "Allow",
         "Action": [
           "codedeploy:RegisterApplicationRevision",
           "codedeploy:GetDeploymentConfig",
           "codedeploy:GetDeployment",
           "codedeploy:GetApplicationRevision",
           "codedeploy:GetApplication",
           "codedeploy:CreateDeployment"
         ],
         "Resource": "*"
       }
     ]
   }
   ```

4. **ECS 사용**

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "",
         "Effect": "Allow",
         "Action": "ecs:*",
         "Resource": "*"
       }
     ]
   }
   ```

5. **역할을 ECS 작업에게 전달 (PassRole)**

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "",
         "Effect": "Allow",
         "Action": "iam:PassRole",
         "Resource": "*",
         "Condition": {
           "StringEqualsIfExists": {
             "iam:PassedToService": "ecs-tasks.amazonaws.com"
           }
         }
       }
     ]
   }
   ```

### [4] CodePipeline 역할 정책 추가

<span style='background-color: #f1f8ff'> **DEV 계정** </span>의 CodePipeline이 **[3]**에서 생성한 <span style='background-color: #ffdce0'> **PROD 계정** </span>의 역할을 사용할 수 있도록 다음 인라인 정책을 추가한다.

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": ["<교차계정 IAM 역할 ARN>"]
  }
}
```

## 3. CodePipeline 설정

CodePipeline에 Deploy 스테이지를 추가한다.

<span style='background-color: #ffdce0'> **PROD 계정** </span>의 CodeDeploy 어플리케이션과 배포 그룹을 기입한다.

해당 어플리케이션과 배포그룹을 찾을 수 없다고 뜰텐데, 무시해도 된다. DEV계정에 있는 것이 아니니 당연히 못찾는다.

이제 파이프라인이 PROD계정의 CodeDeploy로 넘어갈 수 있도록 역할 설정을 해줘야 한다.

AWS CLI를 통해 <span style='background-color: #f1f8ff'> **DEV 계정** </span>의 파이프라인 설정을 json 형태로 가져온다.

```bash
aws codepipeline get-pipeline --name 파이프라인 이름 > pipeline.json
```

pipeline.json을 열고, [1]에서 생성한 고객 관리형 KMS를 사용하도록 설정한다.

```json
    "artifactStore": {
      "type": "S3",
      "location": "codepipeline-ap-northeast-2-845946196676",
      "encryptionKey": {
        "id": "[1]에서 생성한 KMS ARN",
        "type": "KMS"
      }
```

Deploy 스테이지에서 <span style='background-color: #ffdce0'> **PROD 계정** </span>의 역할을 사용하도록 설정한다.

```json
      {
        "name": "Deploy",
          .
          .
          .
            "inputArtifacts": [
              {
                "name": "BuildArtifact"
              }
            ],
            "roleArn": "[3]에서 생성한 교차 계정 IAM 역할 ARN",
            "region": "ap-northeast-2",
            "namespace": "DeployVariables"
          }
```

<span style='background-color: #ffdce0'> **PROD 계정** </span>의 역할을 사용하기 때문에 <span style='background-color: #ffdce0'> **PROD 계정** </span>에 있는 CodeDeploy 어플리케이션에 접근할 수 있다.

맨 아래 위치한 metadata 정보는 지워준다.

```json
"metadata": {
  "pipelineArn": "arn:aws:codepipeline:region:account-ID:pipeline-name",
  "created": "date",
  "updated": "date"
  }
```

다 설정했다면 파일을 **ANSI** 인코딩 타입으로 저장 후 다음 AWS CLI 명령어로 적용한다.

```bash
aws codepipeline update-pipeline --cli-input-json file://pipeline.json
```
