---
title: "[AWS] CodePipeline으로 ECS CI/CD 환경 구성"
date: '2022-03-18 09:30:30'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---

## 1. ECS CodePipeline 개요

도커 이미지 빌드 및 ECS 배포를 자동화해주는 CI/CD 환경을 구축한다.

구성도는 아래와 같다.

![image](https://user-images.githubusercontent.com/60495897/158389314-0a701579-4540-4079-ad72-66a6d7e34d63.png){: width="90%" height="90%" .align-center}

사용자는 이미지로 빌드할 Dockerfile과 빌드 과정이 정의된 buildspec.yml 파일 등을 코드 레포지토리 (여기서는 Github)로 업로드한다.

CodeBuild는 buildspec.yml 에 정의된 내용에 따라 이미지를 빌드하고 ECR에 업로드한다. 

ECS는 업로드된 이미지를 사용해 서비스로 컨테이너를 배포한다.

이러한 과정은 CodePipeline으로 묶여 사용자가 코드를 업데이트할 시 자동으로 실행된다.



## 2. 구성

Node.js 웹애플리케이션을 ECS에 배포해볼것이다.

이번 글에서는 Docekrfile을 빌드하여 ECR로 저장하는 과정을 설명한다.

### 1) ECR 생성

도커 이미지를 저장할 프라이빗 레포지토리를 생성한다.

![image](https://user-images.githubusercontent.com/60495897/157785368-eb4db137-7cfc-4f43-9dab-88c9708fbb97.png){: width="90%" height="90%" .align-center}



### 2) GitHub 레포지토리 생성 및 코드 작성

코드를 저장할 GitHub 레포지토리를 생성한다.

이 글에서 테스트를 위해 사용된 코드 구성은 다음과 같다.



![image](https://user-images.githubusercontent.com/60495897/158284423-ae1038b3-36cc-4467-aca5-a4df0db6dee5.png){: width="90%" height="90%" .align-center}



#### [1] server.js

```javascript
const express= require('express');
const app = express();
const PORT = 8080;

app.get('/', (req, res) => {
    res.send('<h1 style="color:green;">ECS Pipeline Test</h1>\n');
});


app.listen(PORT, () => {
    console.log('Listening on ${PORT}');
});
```

Node.js 와 express로 작성된 간단한 웹서버이다. 8080 포트를 리스닝한다.



#### [2] package.json

```json
{
    "name": "docker_web_app",
    "version": "1.0.0",
    "description": "Node.js on Docker",
    "author": "yuntreee",
    "main": "server.js",
    "scripts": {
        "start": "node server.js"
    },
    "dependencies": {
        "express": "^4.16.1"
    }
}
```



#### [3] Dockerfile

```dockerfile
FROM node:carbon

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 8080

CMD ["npm", "start"]
```

8080포트를 열고 server.js를 실행하도록 하는 Dockerfile이다.



#### [4] buildspec.yml

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin <<<Account ID>>>.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=<<<ECR URI>>>
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":"<<<Container Name>>>","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
artifacts:
    files: imagedefinitions.json
```

CodeBuild는 buildspec.yml 파일에 정의된 내용에 따라 실행된다.

**pre_build** : 빌드 전에 실행할 내용이다. ecr에 로그인하고 환경변수를 설정한다.

**build** : ```docker build ``` 명령어로 이미지를 빌드하고 태그를 붙인다.

**post_build** : 빌드 후에 실행할 내용이다. HASH값으로 생성한 고유한 태그와 latest 태그를 붙여 두번 push한다. latest 태그를 통해 어떤 이미지가 가장 최신에 빌드된 것인지 확인할 수 있다. 

imagedefinitions.json 파일은 컨테이너 배포 시 ECS가 참조하는 파일이다. ```{"name":"컨테이너이름", "imageUri":"이미지URI"}``` 형식이다. 

**artifacts** : buildspec.yml은 artifacts 파일로 imagedefinitions.json 을 내보낸다.



<<<>>> 로 묶여진 내용은 현재 사용하는 정보를 넣으면 된다.  



### 3) ECS 생성

EC2 를 사용하는 ECS 클러스터를 생성한다.

![image](https://user-images.githubusercontent.com/60495897/158296447-688b96f4-55bf-443a-a441-852289f1b0bd.png){: width="90%" height="90%" .align-center}



### 4) CodePipeline 생성

GitHub와 연동 및 CodeBuild를 생성한다.

#### [1] 소스 스테이지

소스 스테이지는 코드가 저장된 GitHub 레포지토리를 선택한다.

![image](https://user-images.githubusercontent.com/60495897/157787592-6a78e599-bc85-4aa6-ba7b-b85880e0ce44.png){: width="90%" height="90%" .align-center}



#### [2] 빌드 스테이지

빌드 공급자를 AWS CodeBuild로 설정 후 빌드 프로젝트를 생성한다.

![image](https://user-images.githubusercontent.com/60495897/157787685-477f5c02-df57-4d7a-8bf4-fe7b84302555.png){: width="90%" height="90%" .align-center}





프로젝트 이름을 작성한다.

![image](https://user-images.githubusercontent.com/60495897/157788229-1fd44cd0-a0bd-477d-93a2-e757120d0f2a.png){: width="90%" height="90%" .align-center}



*관리형 이미지* 선택 후 원하는 환경으로 설정한다.

**도커 이미지를 빌드하거나....** 항목을 활성화한다.

CodeBuild에 사용될 역할은 자동으로 생성되도록 한다.

![image](https://user-images.githubusercontent.com/60495897/157788926-dbf3aabf-2a43-4b45-9bb5-ea863b4ac60c.png){: width="90%" height="90%" .align-center}





buildspec.yml을 사용하여 빌드하도록 한다. 이 값은 디폴트이며, 해당 위치는 소스코드의 루트 디렉토리이다.

![image](https://user-images.githubusercontent.com/60495897/157788446-d11b25f2-7050-496d-9c20-8f181237eb5d.png){: width="90%" height="90%" .align-center}



빌드 스테이지 추가 창으로 돌아와서 생성한 빌드 프로젝트를 선택한다.

![image](https://user-images.githubusercontent.com/60495897/158296854-bb8a92b3-b20c-46da-b31b-21659875da62.png){: width="90%" height="90%" .align-center}



#### [3] 배포 스테이지

배포 스테이지를 추가하기 위해서는 ECS 서비스가 필요하다. 그런데 ECS 서비스를 생성할 때 도커 이미지가 필요하므로 일단 생략한다.



이렇게 파이프라인을 생성을 마치면 자동으로 실행된다.

![image](https://user-images.githubusercontent.com/60495897/158304026-f166e6b0-493c-45ba-a764-a94f3b9eccd6.png){: width="90%" height="90%" .align-center}



```bash
An error occurred (AccessDeniedException) when calling the GetAuthorizationToken operation: User: arn:aws:sts::511914651445:assumed-role/codebuild-yh-ecs-project-service-role/AWSCodeBuild-318774ee-5a00-42f4-b083-6df1a2068ced is not authorized to perform: ecr:GetAuthorizationToken on resource: * because no identity-based policy allows the ecr:GetAuthorizationToken action
```

빌드 과정에서 위와 같은 에러가 발생하는 것은 CodeBuild가 ECR에 접근할 수 있는 권한이 없기 때문이다.

CodeBuild에 부여된 역할에 *AmazonEC2ContainerRegistryFullAccess* 정책을 추가한다.



CodePipeline이 성공적으로 실행되었다면 ECR에 이미지가 저장된 것을 확인할 수 있다.

![image](https://user-images.githubusercontent.com/60495897/158304668-2d11b846-4d6d-4bdd-9cae-be1834b85bcc.png){: width="90%" height="90%" .align-center}
