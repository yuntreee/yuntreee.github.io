---
title: "[AWS] RDS & DMS"
date: '2021-11-07 15:40:14'
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. Amazon RDS 란?

AWS는 Amazon RDS 를 통해 관계형 데이터베이스 서비스를 제공한다. *MySQL, MariaDB, Oracle DB, PostgreSQL, MS SQL, 그리고 Amazon Aurora* 를 운영할 수 있다. 인스턴스가 생성되고 설정에 따라 DB 서버가 구축되는 방식이다. 인스턴스에 직접 접근할 수는 없고, 함께 생성된 **RDS Endpoint**를 사용해 DB를 이용한다. 



### 1) 고가용성

**Mult-AZ 배포** 를 통해 고가용성을 제공한다. 기본 DB 인스턴스를 생성하고 다른 AZ에 있는 인스턴스에 동기적으로 복제한다. DB 인스턴스들은 동일한 Endpoint를 사용하며, Primary DB 인스턴스에 장애가 생길 경우 자동으로 Secondary DB 인스턴스로 링크되어 사용자가 관리할 필요가 없다. 

**스냅샷**을 이용한 백업 및 복원 또한 가능하다. 



### 2) 확장성

읽기 전용 복제본을 생성해 읽기 중심의 워크로드를 처리하기 위해 확장이 가능하다. 읽기 전용 복제본은 Primary DB로부터 비동기적으로 스냅샷을 통해 데이터를 업데이트하며, 다른 리전에도 생성 가능하다. 필요하다면 독립 실행 DB 인스턴스로 승격 가능하다.

읽기 전용 복제본을 생성하기 위해서는 DB인스턴스의 자동백업을 활성화하고, 백업 보존 기간을 0 이외의 수로 설정해야한다.





## 3. DMS (Database Migration Service)

기존 사용하던 데이터베이스를 Amazon RDS로 이주하기 위해 DMS 라는 서비스를 제공한다. (그 반대도 가능하다)

DMS는 이기종 데이터베이스간의 이주도 제공한다.

![image](https://user-images.githubusercontent.com/60495897/140605962-fc1c4cd6-95cd-4f22-94a9-55e825bd51bf.png)

DMS는 복제 인스턴스를 생성하여 Source DB로부터 데이터를 받고 Target DB로 복제한다.

DMS를 사용하여 DB를 백업/복제하면 다운타임이 짧아 기존 DB를 종료하지 않아도 된다.



## 4. RDS & DMS

### 1) DMS

온프레미스의 MySQL DB를 Amazon RDS로 이주하는 작업을 구성한다.

1. Public Access가 가능한 RDS를 생성한다.

2. RDS와 동일한 VPC에 DMS 복제 인스턴스를 생성한다.

3. *소스 / 타겟 엔드포인트* 를 생성한다. *수동으로 액세스 정보 제공* 을 선택하고 각 DB 서버의 IP와 포트, 사용자 이름, 암호를 작성한다.

   연결이 실패되더라도 엔드포인트가 생성되니, 생성 전에 *엔드포인트 연결 테스트* 를 통해 정상 작동을 확인한다.

   ![image](https://user-images.githubusercontent.com/60495897/140633730-f449c911-864c-4eae-a71d-637af804e1e3.png){: width="80%" height="80%" .align-center}

   

   소스 엔드포인트의 스키마를 새로고침하면 어떤 스키마가 작성되어 있는지 확인할 수 있다.

   ![image-20211107145056846](C:\Users\lewis\AppData\Roaming\Typora\typora-user-images\image-20211107145056846.png){: width="80%" height="80%" .align-center}



4. 복제 인스턴스, 소스/타겟 엔드포인트를 사용해 *데이터베이스 마이그레이션 태스크* 를 생성한다.

   어떤 DB의 어떤 테이블을 작업할 것인지 선택 규칙을 작성한다.

   ![image](https://user-images.githubusercontent.com/60495897/140634140-ecb24ab0-22fb-48b8-bfd1-b9846886ab22.png){: width="80%" height="80%" .align-center}



5. 마이그레이션 완료 후 RDS의 DB에 접속하여 DB가 복제되었음을 확인할 수 있다.

   ```bash
   $ mysql -u<사용자이름> -p -h <RDS Endpoint>
   ```



### 2) Read Replica

![image](https://user-images.githubusercontent.com/60495897/140634966-2cf954cf-0b6d-4d11-b0c7-3bbea4c45f22.png){: width="80%" height="80%" .align-center}

읽기 전용 복제본을 생성하여 접속해본다. 사용자와 패스워드는 원본과 동일하다.

읽기 전용 복제본에서는 SELECT 구문만 실행 가능하다. 

원본에서 데이터를 변경하면 복제본에도 업데이트되는 것을 확인할 수 있다.

