---
title: "[CentOS 8] 웹 가상호스트 (Virtual Host)"
date: '2021-09-03 00:51:14'
categories: Linux
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 가상호스트 (Virtual Host) 란

하나의 IP로 다수의 웹서버를 호스팅할 때 사용한다.



## 2. 가상호스트 웹서버 구축

두개의 다른 도메인을 사용하여 가상호스트를 구축한다.

### 1) httpd.conf

```bash
# vim /etc/httpd/conf/httpd.conf
-------------------------------------------------
include /etc/httpd/conf/vhost.conf
<Directory "/home">
	AllowOverride None
	Require all granted
</Directory>
```

### 2) vhost.conf 

```bash
# vim /etc/httpd/conf/vhost.conf
--------------------------------------------------
<VirtualHost *:80>
	DocumentRoot /home/chul
	ServerName chul.com
	ServerAlias	www.chul.com
</VirtualHost>

<VirtualHost *:80>
	DocumentRoot /home/jeong
	ServerName jeong.com
	ServerAlias www.jeong.com
</VirtualHost>
```



dns 정보는 따로 설정해준다. chul.com과 jeong.com을 동일하게 192.168.111.100 으로.