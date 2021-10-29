---
title: "[AWS] CodePipeline"
date: '2021-10-29 01:00:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. CodePipeline 이란?

CodePipeline이란 간단히 말해 CodeCommit, CodeBuild, CodeDeploy를 하나의 프로세스로 통합시켜주는 CI/CD 도구이다. 코드 변경이 있을 때마다 사용자가 사전에 정의한 릴리스 모델을 기반으로 빌드, 테스트, 배포 단계를 자동화한다. 이를 통해 어플리케이션의 변경 사항이나 새로운 기능을 신속하고 안정적으로 제공할 수 있다.

![image](https://user-images.githubusercontent.com/60495897/139240351-5e8d5b51-1b1c-4041-a18a-92fcbca76db5.png)

소스코드는 CodeCommit, Github, ECR, S3 에서 가져올 수 있다.



## 2. CodePipeline 구성

이전 포스트들을 ([Cloud9&CodeCommit](https://yuntreee.github.io/aws/aws_C9&CodeCommit/), [CodeBuild&CodeDeploy](https://yuntreee.github.io/aws/aws_CodeBuild&CodeDeploy/)) 참고하여 Cloud9 -> CodeCommit -> CodeBuild -> CodeDeploy 의  프로세스를 거치는 어플리케이션을 구성한다.

구성된 각 단계들을 CodePipeline으로 묶고 파이프라인을 실행하여 확인한다.



어플리케이션에 변경사항이 있다면, Cloud9에서 코드 변경 후 CodeCommit 레포지토리로 push한다. CodePipeLine은 레포지토리가 업데이트된 것을 감지하고 자동으로 재배포한다.

혹은 Auto Scailing 그룹에 의해 인스턴스가 증가하여도 자동으로 재배포한다.

즉, Pipeline을 한번 만들어놓으면 레포지토리에 코드를 업데이트하는것 외의 작업들이 자동화된다.
