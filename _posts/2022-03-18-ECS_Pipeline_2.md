---
title: "[AWS] CodePipeline으로 ECS CI/CD 환경 구성(2)"
date: '2022-03-18 09:30:30'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---

## 1. 구성

이번 글에서는 ECS 서비스로 빌드된 도커 이미지를 배포하는 과정을 CodePipeline에 추가한다.



### 1) ALB 생성 및 EC2 보안그룹 수정

동일한 어플리케이션이 실행중인 복수개의 컨테이너에 트래픽을 분산하기 위해 ALB를 사용한다.

80포트를 사용하여 호스트EC2로 HealthCheck하는 대상그룹을 생성하고, 80포트를 리스닝하는 ALB에 연결한다.



ALB로부터 모든 트래픽을 허용하도록 EC2의 보안그룹을 수정한다. 그 이유는 동적 포트 매핑을 사용하기 위함으로, 다음 목차에서 추가로 설명한다.



### 2) ECS 서비스 생성

이제 빌드된 컨테이너 이미지가 있으므로 ECS 서비스를 생성할 수 있다.

**[1] 작업 정의 생성**

시작유형을 EC2로 설정하여 작업 정의를 생성한다. (Fargate도 상관 없지만 비교적 비싸다)

아래와 같이 작업정의에 컨테이너를 추가한다.

![image](https://user-images.githubusercontent.com/60495897/158305217-f621cadf-f733-4dfd-a973-97b4e055f9ed.png){: width="90%" height="90%" .align-center}

이미지는 ECR URI를 입력한다. 해당 레포지토리에서 latest태그가 붙은(가장 최신인) 이미지로 컨테이너가 생성된다.

<div class="notice--primary" markdown="1">

호스트 포트는 0으로 설정한다. 이는 동적 포트 매핑(Dynamic Port Mapping)을 사용하기 위함이다.

동적 포트 매핑은 컨테이너에 매핑되는 호스트 포트를 고정하지 않는다. ECS가 랜덤한 호스트 포트를 알아서 컨테이너로 매핑시켜준다. 

이는 후에 새로 업데이트된 이미지를 배포할 때 필요하다. 무중단 롤링 업데이트시 기존 작업을 유지하는 동안 새 이미지로 작업을 생성한다. 예를 들어 서비스에서 실행중인 1개의 작업을 업데이트하는 과정은 *1개(OLD)->2개(OLD+NEW)->1개(NEW)* 으로 진행된다.

그런데 복수개의 작업이 호스트의 같은 포트를 사용할 수는 없다.

 만약 호스트:80->컨테이너:8080 으로 고정 포트 매핑을 사용하는 상황이라면, 2개(OLD+NEW)의 작업이 동시에 호스트80포트를 사용할 수 없기 때문에 에러가 발생한다. 

따라서 호스트의 포트를 고정하지 않아야 무중단 배포가 가능하다.

</div>



**[2] 서비스 생성**

시작유형이 EC2 인 서비스를 구성한다.

![image](https://user-images.githubusercontent.com/60495897/158308491-8aeab675-9780-4b6c-bd2d-1fdc7602cf08.png){: width="90%" height="90%" .align-center}





생성해놓은 ALB에 서비스를 연결한다.

![image](https://user-images.githubusercontent.com/60495897/158308704-7eb0ce12-b026-426d-b868-b84ed458a1cd.png){: width="90%" height="90%" .align-center}

![image](https://user-images.githubusercontent.com/60495897/158308784-5a371bbd-752e-4da4-b2cc-8eb3d1665a2d.png){: width="90%" height="90%" .align-center}



![image](https://user-images.githubusercontent.com/60495897/158308956-90817dcc-abc1-41d2-8eaf-626d18a4d30e.png){: width="90%" height="90%" .align-center}

서비스가 생성되어 작업이 실행되고, 연결해둔 ALB DNS 로 접속하면 Node.js 로 동작하는 웹 화면을 볼 수 있다.

![image](https://user-images.githubusercontent.com/60495897/158315599-48341db3-aa4b-42cb-b182-aaba50ef7015.png){: width="90%" height="90%" .align-center}





### 3) ECS 배포 생성

서비스를 생성했으니 CodePipeline에 배포를 추가할  수 있다.

파이프라인 편집으로 이동하여 CodeBuild 다음 스테이지에 ECS를 사용한 배포를 추가한다.

![image](https://user-images.githubusercontent.com/60495897/158391200-8f2551e2-8aae-484e-bb70-a081a8b3611c.png){: width="90%" height="90%" .align-center}

이미지 정의 파일은 buildspec.yml에서 설정한대로 *imagedefinitions.json* 으로 설정한다.

테스트를 위해 GitHub 코드를 업로드하면 파이프라인이 실행되어 업데이트한 웹 페이지가 배포된다.

![image](https://user-images.githubusercontent.com/60495897/158395452-f239d49c-0b82-4dd9-947f-977fbb248a48.png){: width="90%" height="90%" .align-center}





![image](https://user-images.githubusercontent.com/60495897/158394857-019fb89c-983a-4b8d-9a78-a022ba2f4922.png){: width="90%" height="90%" .align-center}

ECR 레포지토리를 확인하면 새로 빌드된 이미지가 latest태그로 저장되어 있음을 확인할 수 있다.

![image](https://user-images.githubusercontent.com/60495897/158395952-12da7e18-fef2-412d-98d3-69edcea137bf.png){: width="90%" height="90%" .align-center}



ECS 서비스 이벤트를 확인하면 위에서 언급한 순서대로 (1개 -> 2개 -> 1개) 배포가 진행됨을 확인할 수 있다.

![image](https://user-images.githubusercontent.com/60495897/158396482-6e5c10bf-2212-489e-920f-fa6bbe577a8a.png){: width="90%" height="90%" .align-center}