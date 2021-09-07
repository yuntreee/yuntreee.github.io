---
title: "[CentOS 8] iSCSI"
date: '2021-09-04 19:42:14'
categories: Linux
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. iSCSI 란

IP 기반 네트워크로 SCSI 명령을 전달하여 블록 단위의 데이터 전송을 한다. 

NAS 가 파일 공유하는 목적이라면, iSCSI는 스토리지 자체를 1대1로 원격에서 사용하기 위함이다.



서버 역할의 iSCSI Target과 클라이언트 역할의 iSCSI Initiator 로 구분된다.



## 2. iSCSI 구현

현재 서버에는 논리 볼륨 LV1(15GB) LV2(10GB) LV3(5GB) 가 구성되어 있는 상태이다.

LV1과 LV2를 하나의 타겟, LV3를 하나의 타겟으로 iSCSI를 구성한다.

### 1) Server

```bash
# dnf -y install targetcli*
# systemctl start target
# firewall-cmd --permanent --add-port=3260/tcp
# firewall-cmd --reload
```



공유할 논리 볼륨 (혹은 물리 볼륨)을 backstores에 block으로 지정한다. 클라이언트 입장에서 이 block은 보이지 않는다.

```bash
# targetcli
/> /backstores/block create data1 /dev/myVG/LV1
/> /backstores/block create data2 /dev/myVG/LV2
/> /backstores/block create data3 /dev/myVG/LV3
```



서버쪽 IQN (iSCSI Qualified Name)을 생성하고 data를 타겟에 LUN (Logical Unit Number)으로 인식시킨다.

**IQN 형식** : iqn.YYYY-MM.Naming_Auth(도메인 역순):설명

**LUN** : iSCSI Target이 제공하는 논리적 SCSI 장치로, backstore과 매칭된다.

**tgp (target portal group)** : initiator들이 target에 연결될 수 있도록 하는 설정들의 그룹

```bash
/> /iscsi/ create iqn.2021-09.com.chul.master:tgt1

/> /iscsi/iqn.2021-09.com.chul.master:target/tpg1
> luns/ create /backstores/block/data1
> luns/ create /backstores/block/data2
> set attribute authentication=1			(CHAP 인증 적용)
```



접속할 클라이언트 iqn을 ACL에 등록하고 id와 passwd를 설정한다.

```bash
> acls/ create iqn.2021-09.com.chul:slave
> cd acls/iqn.2021-09.com.chul:slave
> set auth userid=slave
> set auth password=p@ssw0rd1234
> / saveconfig
> exit

# systemctl restart target
```



동일한 방식으로 LV3로 tgt2를 구성하면 된다.



### 2) Client

tgt1을 연결해본다.

IQN을 설정하고 iSCSI 설정을 수정한다.

```bash
# vim /etc/iscsi/initiatorname.iscsi
---------------------------------------------------------------------
InitiatorName=iqn.2021-09.com.chul:slave

# systemctl start iscsid

# vim /etc/iscsi/iscsid.conf
---------------------------------------------------------------------
node.session.auth.authmethod = CHAP
node.session.auth.username = slave
node.session.auth.password = p@ssw0rd1234

# systemctl restart iscsid
```



master서버의 포탈이 열려있는지 확인하고 연결한다.

```bash
# iscsiadm --mode discoverydb --type sendtargets --portal 192.168.111.100 --discover
# iscsiadm --mode node --targetname iqn.2021-09.com.chul.master:tgt1 --portal 192.168.111.100:3260 --login

# fdisk -l
```

iSCSI로 연결된 하드디스크 두개가 확인된다. 파티셔닝 후 사용하면 된다.