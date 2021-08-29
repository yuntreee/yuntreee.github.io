---
title: "[CentOS 8] 사용자/그룹 관리"
date: '2021-08-28 13:19:14'
categories: Linux
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 사용자 계정 정보

### 1) /etc/passwd

7개의 필드로 되어있다.

| 1      | 2        | 3    | 4    | 5        | 6           | 7       |
| ------ | -------- | ---- | ---- | -------- | ----------- | ------- |
| 계정명 | 패스워드 | UID  | GID  | 계정정보 | 홈 디렉토리 | 사용 쉘 |

패스워드 정보는 /etc/shadow 에 암호화되어 저장된다.



### 2) /etc/shadow

9개의 필드로 되어있다.

| 필드         | 설명                                                         |
| ------------ | ------------------------------------------------------------ |
| Login Name   | 사용자의 계정명이다                                          |
| Encrypted    | $algorithm_id$salt$encrypted_password <br>혹은 *, !, 빈값이 설정되어 있을 수 있다. <br>* * :  패스워드가 잠긴 상태. 로그인 불가. <br>!  : 패스워드가 잠긴상태. 로그인 불가. 또는 패스워드가 설정되지 않은 상태<br>공백 : 패스워드가 설정되지 않았지만 로그인이 가능한 상태 |
| Last Changed | 패스워드를 변경한 마지막 날<br>1970년 1월 1일을 기준으로 일수를 표시 |
| Minimum      | 패스워드 변경은 최소 이 기간 이후에 가능하다                 |
| Maximum      | 패스워드 만료 기간이다                                       |
| Warn         | 패스워드 만료 이전에 경고할 경고 일수이다                    |
| Inactive     | 패스워드가 만료된 이후 계정이 잠기기 전까지 비활성화 기간이다. |
| Expire       | 계정 만료 기간이다.                                          |
| Reserved     | 예약필드                                                     |



### 3) /etc/default/useradd

사용자 계정 생성시 기본 설정되는 정보이다.

| 항목              | 설명                                                         |
| ----------------- | ------------------------------------------------------------ |
| GROUP             | 기본 GID                                                     |
| HOME              | 사용자의 홈 디렉토리가 놓여질 디렉토리                       |
| INACTIVE          | 패스워드 사용기간이 지난 계정의 로그인을 막는 유예기간<br>0: 유예기간 없음<br>-1: 설정되지 않음 |
| EXPIRE            | 계정의 만료일                                                |
| SHELL             | 기본 사용 쉘                                                 |
| SKEL              | 사용자 생성시 제공되는 파일 및 디렉토리가 있는 디렉토리      |
| CREATE_MAIL_SPOOL | 사용자 생성시 메일 파일을 생성할 것인지<br>yes일 시 "var/spool/mail/사용자명" 에 메일파일 생성 |



## 2. 사용자 추가

```useradd``` 명령어를 사용한다. 주요 옵션은 다음과 같다.

### 1) useradd

#### useradd <사용자명>

```shell
[root@master ~]# useradd user1
```

별다른 옵션 없이 사용자를 추가한다. /etc/default/useradd 내용대로 사용자가 추가된다. 

추가된 사용자는 /etc/passwd 에서 확인 가능하다.

```shell
[root@master ~]# tail -1 /etc/passwd
user1:x:1001:1001::/home/user1:/bin/bash
```

자신의 id와 동일한 그룹에 추가된 것을 확인할 수 있다.



#### useradd -g <gid> <사용자명>

```-g``` 옵션은 추가될 사용자가 소속될 그룹을 지정한다. 물론 기존에 존재하는 gid여야 한다.

```shell
[root@master ~]# useradd -g 1000 user1
[root@master ~]# tail -1 /etc/passwd
user1:x:1001:1000::/home/user1:/bin/bash
```



#### useradd -G <gid> <사용자명>

```-G``` 옵션은 사용자가 소속될 2차 그룹을 지정한다.



#### useradd -u <uid> <사용자명>

```-u``` 옵션은 사용자의 uid를 설정한다. 직접 설정하지 않을 때에는 /etc/passwd에 존재하는 마지막 uid보다 1 높은 값이 된다.

```shell
[root@master ~]# useradd -u 1100 user1
[root@master ~]# tail -1 /etc/passwd
user1:x:1100:1100::/home/user1:/bin/bash

[root@master ~]# useradd user2
[root@master ~]# tail -2 /etc/passwd
user1:x:1100:1100::/home/user1:/bin/bash
user2:x:1101:1101::/home/user2:/bin/bash
```

user1 uid를 1100으로 설정하여 생성하고, 뒤이어 user2를 생성하면 uid가 1101로 부여됨을 확인할 수 있다.



#### useradd -d <dir> <사용자명>

```-d``` 옵션은 사용자의 홈 디렉토리를 설정해준다. 

```shell
[root@master ~]# useradd -d /home/newdir user1
[root@master ~]# tail -1 /etc/passwd
user1:x:1001:1001::/home/newdir:/bin/bash
```



#### useradd -e <만기일> <사용자명>

```-e``` 옵션은 계정의 만기일을 지정한다. 만기일은 ```YYYY-DD-MM``` 형식이다. 

```shell
[root@master ~]# useradd -e 2022-12-12 user1

[root@master ~]# tail -1 /etc/shadow
user1:!!:18864:0:99999:7::19338:
```

설정된 계정 만기일은 /etc/shadow 의 8번째 필드에서 확인 가능하다. 단, YYYY-MM-DD 형식이 아니라 1970년 1월 1일로부터 만기일자까지의 일수로 저장된다.



#### useradd -f <날짜수> <사용자명>

```-f``` 옵션은 사용자 패스워드 만기일을 지정한다. 

```shell
[root@master ~]# userdel -r user1

[root@master ~]# useradd -f 5 user1
[root@master ~]# tail -1 /etc/shadow
user1:!!:18864:0:99999:7:5::
```

설정된 패스워드 만기일은 /etc/shadow 의 7번째 필드에서 확인 가능하다.



### 2) passwd

```passwd```는 계정의 암호와 관련된 명령어다.



#### passwd <사용자명>

사용자 계정의 패스워드를 설정한다.



#### passwd -l <사용자명>

사용자 계정을 잠근다.



#### passwd -u <사용자명>

사용자 계정을 잠금 해제한다.



#### passwd -n <숫자> <사용자명>

사용자의 패스워드 최소 변경 기간을 설정한다. (/etc/shadow의 Minimum 필드를 수정)



#### passwd -x <숫자> <사용자명>

사용자의 패스워드 만료일을 설정한다. (/etc/shadow의 Maximum 필드를 수정)



#### passwd -w <숫자> <사용자명>

사용자 패스워드 변경 만료일 몇일 전에 경고메세지를 보여줄 것인지 설정한다. (/etc/shadow의 Warning 필드를 수정)



#### passwd -i <숫자> <사용자명>

패스워드 만료일 몇일 후 계정이 잠길 것인지 설정한다. (/etc/shadow의 Inactive 필드 수정)



#### passwd -e <사용자명>

사용자 계정을 강제로 만료시켜 패스워드를 변경하도록 한다.



### 3) groupadd

그룹을 추가한다. 그룹의 정보는 /etc/group 에서 확인할 수 있다.



#### groupadd -g <GID> <그룹명>

지정한 GID로 그룹을 생성한다. ```-g``` 옵션을 빼는 경우 자동으로 gid가 설정된다.



## 2. 사용자 정보 변경

### 1) usermod

```usermod``` 는 사용자 계정의 여러 정보를 변경할 수 있다.



#### usermod -l <새로운계정> <기존계정명>

계정의 이름을 바꾼다.



#### usermod -dm <디렉토리> <사용자명>

사용자의 홈 디렉토리를 변경한다. ```-d``` 옵션은 디렉토리 변경을, ```-m``` 옵션은 기존 디렉토리에서 새로운 디렉토리로 파일을 옮기는 옵션이다.



#### usermod -u <UID> <사용자명>

사용자의 UID를 변경한다.



#### usermod -g <GID> <사용자명>

사용자의 그룹을 변경한다. ```-G``` 옵션은 2차 그룹을 변경한다.



#### usermod -e <YYYY-DD-MM> <사용자명>

사용자의 계정 만료일을 변경한다.



#### usermod -f <숫자> <사용자명>

패스워드 만료일 몇일 후 계정이 잠길 것인지 변경한다.



#### usermod -L <사용자명>

사용자의 계정을 잠근다.



#### usermod -U <사용자명>

계정 잠금을 해제한다.



### 2) chage

```chage```는 패스워드의 에이징에 관한 명령어이다. 물론, passwd 와 usermod 명령으로도 에이징을 설정할 수 있다.

| 옵션 | 설명                                                         |
| ---- | ------------------------------------------------------------ |
| -d   | 최근 패스워드를 바꾼 날짜를 수정한다. YYYY-MM-DD 혹은 1970/01/01 이후로 지난 날짜로 설정 |
| -E   | 계정 만료일을 설정한다.                                      |
| -m   | 패스워드 최소 사용 기간을 설정한다.                          |
| -M   | 패스워드 최대 사용 기간을 설정한다.                          |
| -W   | 패스워드 변경 만료일 몇일 전에 경고메세지를 보여줄 것인지 설정한다. |



### 3) groupmod

그룹의 정보를 변경한다. 



#### groupmod -g <GID> <그룹명>

그룹의 GID를 변경한다



#### groupmod -n <새로운그룹명> <기존그룹명>

그룹명을 변경한다.



## 3. 사용자 제거

### 1) userdel

사용자 제거에는 ```userdel``` 명령어를 사용한다.

#### userdel -r <사용자명>

사용자의 홈 디렉토리까지 삭제한다



#### userdel -f <사용자명>

사용자가 로그인 중이여도 강제로 삭제한다.



## 4. 파일 소유주 변경

#### chown -R <사용자명>:<그룹명> <파일이름>

파일의 소유주나 소유그룹을 변경한다. ```-R``` 옵션은 하위 파일까지 변경내용을 적용한다.
