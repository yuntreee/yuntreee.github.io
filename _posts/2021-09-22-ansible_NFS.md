---
title: "[Ansible] NFS 설치"
date: '2021-09-22 23:14:14'
categories: Ansible
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## NFS 

### NFS yaml

```yaml
---
#-------------NFS 서버 설정-------------#
- name: Setup NFS Server
  hosts: localhost

  tasks:
    - name: make nfs directory
      file:
        path: /home/vagrant/nfs_shared
        mode: 0777
        state: directory
    
    - name: configure /etc/exports
      become: yes
      lineinfile:
        path: /etc/exports
        line: /home/vagrant/nfs_shared 192.168.0.0/24(rw,sync)

    - name: restart nfs
      become: yes
      service:
        name: nfs
        state: restarted

#-------------NFS 클라이언트 설정-------------#
- name: Setup NFS Client
  hosts: CentOS, Ubuntu
  
  tasks:
    - name: make nfs directory
      file:
        path: /home/vagrant/nfs
        state: directory
    
    - name: mount nfs directory
      mount:
        path: /home/vagrant/nfs
        src: 192.168.0.60:/home/vagrant/nfs_shared
        fstype: nfs
        opts: nfsvers=3
        state: mounted
```



### 모듈 설명

```file```

파일을 생성 및 삭제할 때 사용되는 모듈이다.

state 파라미터에는 6가지 종류가 있다.

file: 디폴트 값이다. 파일 소유자, 권한 변경 등을 할 수 있다. 해당 파일이 존재하지 않으면 생성하지 않는다.

directory: 디렉토리가 존재하지 않을 경우 recursive하게 생성한다.

hard: 하드링크를 생성한다. src와 dest 파라미터를 적어야 한다.	

link: 심볼릭 링크를 생성한다. src와 dest 파라미터를 적어야 한다.

touch: 파일을 생성한다.

absent: 해당 파일을 삭제한다. 디렉토일 경우 recursive하게 삭제한다.



```lineinfile```

파일 내용을 줄 단위로 수정한다.

state: line을 지울지 추가할지 선택한다. absent, present가 있으며, 기본값은 present 이다.

line: state가 present일 경우 사용되는 줄이다. 기본적으로는 파일의 마지막 행에 추가된다.

regexp: 정규표현식을 적는다. state=absent일 경우 표현식에 맞는 모든 줄이 삭제된다. state=present일 경우 표현식에 맞는 가장 마지막 줄만 작업에 해당된다.

backrefs: 기본값은 no이다. yes일 경우 regexp에 해당하는 줄이 line으로 대체된다.



```mount```

마운트할 때 사용된다. state 옵션으로는 5가지가 있다.

mounted : 마운트하고 */etc/fstab* 에 등록한다. 마운트포인트가 존재하지 않는다면 생성한다. 

present: */etc/fstab* 에 등록하고 마운트하지는 않는다.

umounted: 마운트를 해제하지만 */etc/fstab* 의 내용은 삭제하지 않는다.

absent: 완전히 지운다. 마운트를 해제하고, */etc/fstab* 의 내용을 삭제하고, 마운트포인트도 제거한다.

remounted: 다시 마운트한다.







