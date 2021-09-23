---
title: "[Ansible] Nginx 설치"
date: '2021-09-23 00:58:14'
categories: Ansible
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## Nginx 설치 및 삭제

### 설치 yaml

```yaml
---
#--------CentOS--------#
- name: Install nginx on CentOS
  hosts: CentOS
  become: yes

  tasks:
    - name: install epel-release
      yum: name=epel-release state=latest
    
    - name: install nginx
      yum: name=nginx state=present
    
    - name: start nginx
      service: name=nginx state=started

#--------Ubuntu--------#
- name: Install nginx on Ubuntu
  hosts: Ubuntu
  become: yes

  tasks:
    - name: install nginx
      apt: pkg=nginx state=present update_cache=yes

```



### 삭제 yaml

```yaml
---
#--------CentOS--------#
- name: Remove nginx on CentOS
  hosts: CentOS
  become: yes

  tasks:
    - name: remove epel-release
      yum: name=epel-release state=absent
    - name: remove nginx
      yum: name=nginx state=absent

#--------Ubuntu--------#
- name: Remove nginx on Ubuntu
  hosts: Ubuntu
  become: yes

  tasks:
    - name: remove nginx
      apt: name=nginx state=absent
```





### 모듈 설명

```become```

노드 내에서 플레이북 실행 시 어떤 특정 사용자 권한으로 실행할 지 결정한다. 기본값은 false/no 이다.



```become_user```

``become`` 을 yes로 하고 실행할 사용자 권한 이름을 정한다. 기본값은 root이다.



```yum```, ```apt``` 

CentOS는 yum, Ubuntu는 apt를 사용한다.

present=설치, absent=삭제, latest=저장소 내의 최신버전 설치를 의미한다.



```service```

CentOS의 **systemctl** 같은거다. started=실행, stopped=중지, restarted=재실행 을 의미한다.