---
title: "[AWS] EKS CodePipeline 구축"
date: '2022-01-04 00:01:30'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. CodePipeline을 활용한 EKS 배포

Github (혹은 CodeCommit) -> CodeBuild -> ECR -> EKS 로 이어지는 파이프라인을 구축하여 도커 이미지의 빌드와 배포를 자동화한다.



### 1) EKS 클러스터 생성

```bash
eksctl create cluster 
--name [클러스터 이름] 
--version 1.18 
--region ap-northeast-2 
--nodegroup-name linux-noded 
--nodes [노드 개수] 
--nodes-min [최소 노드 개수] 
--nodes-max [최대 노드 개수]
--ssh-access --ssh-public-key [키페어 명] 
--node-type t3.medium 
--managed 
```





### 2) Github Repository 생성

CodePipeline과 연동할 저장소를 생성한다.

필요한 코드는 다음과 같다.

![image](https://user-images.githubusercontent.com/60495897/147954720-7c172f84-42b4-4251-81df-adf9d0975f9a.png)

buildspec.yaml은 CodeBuild에서 어떻게 빌드할 것인가 정의하는 파일이다.

Dockerfile은 생성할 도커 이미지이며, index.html은 이미지에 쓰인다.

EKS 디렉토리에는 배포할 디플로이먼트와 서비스(로드밸런서) yaml 파일이 있다.



```yaml
# buildspec.yaml
version: 0.2
phases:
  install:
    runtime-versions:
      docker: 18
    commands:
      - curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
      - chmod +x ./kubectl
      - mv ./kubectl /usr/local/bin/kubectl
      - mkdir ~/.kube
      - aws eks --region ap-northeast-2 update-kubeconfig --name eks-demo
      - kubectl get pod -n kube-system
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/
  build:
    commands:
      - echo Building the Docker image
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG

  post_build:
    commands:
      - AWS_ECR_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
      - DATE='date'
      - echo Build completed on $DATE
      - sed -i.bak 's#AWS_ECR_URI#'"$AWS_ECR_URI"'#' ./EKS/Deployment.yaml
      - sed -i.bak 's#DATE_STRING#'"$DATE"'#' ./EKS/Deployment.yaml
      - kubectl apply -f ./EKS/Deployment.yaml
      - kubectl apply -f ./EKS/Service.yaml

```

commands : CodeBuild가 실행되는 서버에 kubectl을 설치한다.

pre_build.commands : 도커 이미지파일을 업로드하고 다운받을 수 있는 ECR에 로그인한다.

build.commands : 도커 이미지를 생성한 후 ECR로 업로드한다.

post_build.commands : kubectl 명령어를 사용하여 EKS 클러스터에 디플로이먼트와 서비스를 배포한다.



```dockerfile
# Dockerfile
FROM ubuntu:18.04
RUN apt-get -y update
RUN apt-get -y install apache2
COPY index.html /var/www/html
EXPOSE 80
CMD apachectl -DFOREGROUND
```

간단한 아파치 서버를 가동하는 도커파일이다.



```html
# index.html
<h1> TEST EKS PIPELINE </h1>
```



```yaml
# Deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eks-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: eks-apache
  template:
    metadata:
      labels:
        app: eks-apache
    spec:
      containers:
      - name: apache
        image: AWS_ECR_URI
        ports:
        - containerPort: 80
        imagePullPolicy: Always
```



```yaml
# Service.yaml
apiVersion: v1
kind: Service
metadata: 
  name: loadbalancer-service
spec:
  type: LoadBalancer
  selector:
    app: eks-apache
  ports:
  - protocol: TCP
    nodePort: 30080
    port: 80
```

AWS의 ELB와 연동될 것이다.



### 3) ECR 생성

도커 이미지 저장소인 ECR을 생성한다.



### 4) CodeBuild 생성

도커파일을 빌드하고 EKS로 배포할 수 있는 CodeBuild를 생성한다.

![image](https://user-images.githubusercontent.com/60495897/147952711-a4e810fd-7c2b-432d-8fb5-59877dc75d14.png){: width="70%" height="70%" .align-center}

 생성한 Github 저장소에 연결한다.



![image](https://user-images.githubusercontent.com/60495897/147952804-c0a263d1-bff2-47f7-8f2a-2b624826c6af.png){: width="70%" height="70%" .align-center}

CodeBuild로 도커 이미지를 생성할 것이기 때문에 *Managed image* 선택 후 *Privileged* 를 체크한다.



EKS가 있는 VPC를 선택 후 Private 서브넷들을 고른다. 해당 Private 서브넷은 모두 외부와 통신할 수 있도록 NAT Gateway가 연결되어 있어야 한다. *Validate VPC Settings*로 유효한 서브넷인지 검증한다.



![image](https://user-images.githubusercontent.com/60495897/147952362-17382f18-888c-47d7-a0f0-76c98dbb7d52.png){: width="70%" height="70%" .align-center}

buildspec.yaml에서 사용될 환경변수를 설정한다.

IMAGE_REPO_NAME : 빌드된 이미지가 저장될 저장소로, 생성한 ECR 이름을 기입한다.

IMAGE_TAG : 빌드된 이미지에 붙여질 태그이다.



![image](https://user-images.githubusercontent.com/60495897/147952964-fb261839-72bd-4b70-b049-9dfb7e11a041.png){: width="70%" height="70%" .align-center}

Buildspec 파일의 이름을 설정한다.



### 5) CodePipeline 생성

CodePipeline을 생성한다. . Source Stage는 Github, Build Stage는 CodeBuild로 설정한다.

Deploy Stage는 EKS Fargate가 해주므로 생략한다.



### 6) CodeBuild 접근 허용

EKS 클러스터에 CodeBuild가 접근할 수 있도록 허용해야 kubectl 명령이 가능하다.

클러스터의 aws-auth 라는 configmap을 추출한 후 다음과 같이 수정한다.

```bash
kubectl get configmaps aws-auth -n kube-system -o yaml > aws-auth.yaml
```

```yaml
# aws-auth.yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::<계정번호>:role/<eks노드 역할>
      username: system:node:{{EC2PrivateDNSName}}
# --------------------------- 추가 내용 --------------------------- #
	- rolearn: arn:aws:iam::<계정 번호>:role/<CodeBuild Role 이름>
      username: <CodeBuild Role 이름>
      groups: 
        - system:masters
# ---------------------------------------------------------------- #

kind: ConfigMap
metadata:
  creationTimestamp: "2022-01-03T13:06:35Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:mapRoles: {}
    manager: vpcLambda
    operation: Update
    time: "2022-01-03T13:06:35Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "1518"
  selfLink: /api/v1/namespaces/kube-system/configmaps/aws-auth
uid: dce3e993-17d1-4a5d-acb4-1909c7150aef

```

추가내용의 rolearn은 CodeBuild 역할 ARN에서 /service-role 경로를 제거한 것이다.



## 2. 결과 확인

CodePipeline 릴리즈 후 결과를 확인한다.

```kubectl get pods,deploy,svc``` 로 배포 결과를 확인 후, 생성된 ELB 주소로 웹서비스에 접근해본다.

![image](https://user-images.githubusercontent.com/60495897/147955755-489474ef-76df-4f4f-8148-c92ead0f57b5.png){: width="70%" height="70%" .align-center}



ECR에서 생성한 도커 이미지가 업로드되어 있음을 확인할 수 있다.

![image](https://user-images.githubusercontent.com/60495897/147955829-20ae18d3-c58c-4eca-b044-7a17c6f46479.png){: width="70%" height="70%" .align-center}