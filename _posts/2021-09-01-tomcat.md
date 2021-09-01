---
title: "[CentOS 8] 톰캣 (Tomcat) 서버"
date: '2021-09-01 02:26:14'
categories: Linux
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. Tomcat 이란

Apache는 html, css 와 같은 정적 데이터만 지원한다. jsp나 php같은 동적 프로그래밍 언어를 해석하기 위해서는 Tomcat과 같은 별도의 어플리케이션이 필요하다.

톰캣은 java 코드를 사용해 html 페이지를 동적으로 생성해준다. 아파치로 생성한 웹 서버와 톰캣으로 생성한 웹 컨테이너를 결합하여 동적 데이터는 아파치가 처리하고 정적 데이터는 톰캣에게 넘겨 결과를 받아 다시 클라이언트에게 응답한다.

![image](https://user-images.githubusercontent.com/60495897/131690146-f19b4f5c-466e-409c-bafd-d76627203203.png)



아파치에는 **mod_jk** 모듈이 아파치 웹서버와 톰캣 서버와의 통신을 지원한다.

혹은 아파치의 기본 모듈인 **mod_proxy** 와 **mod_proxy_ajp** 를 사용하여 톰캣 서버와 연동할 수 있다. 단, 이 모듈은 기본적인 부하분산 기능만을 지원한다.



## 2. WAS 구현

192.168.111.200 서버에서 진행한다.

### 1) java 설치 

```dnf -y install java-1.8.0-openjdk-devel.x86_64```

**환경변수 설정**

```bash
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.302.b08-0.el8_4.x86_64
CATALINA_HOME=/usr/local/tomcat8
CLASSPATH=.:$JAVA_HOME/lib/tools.jar:$CATALINA_HOME/lib/jsp-api.jar:$CATALINA_HOME/lib/servlet-api.jar
PATH=$PATH:$JAVA_HOME/bin:$CATALINA_HOME/bin
export JAVA_HOME PATH CLASPATH
```



### 2) Tomcat 설치

```wget http://archive.apache.org/dist/tomcat/tomcat-8/v8.5.70/bin/apache-tomcat-8.5.70.tar.gz```

압축 풀고 **/usr/local/tomcat8** 로 이동한다.

톰캣 설정은 **conf/server.xml** 에서 한다. 한글을 쓸거면 여기에 **URIEncoding="UTF-8"** 을 추가한다.

톰캣 서비스 시작은 **bin/startup.sh**, 종료는 **bin/shutdown.sh** 을 실행한다.

톰캣의 기본 웹문서 경로는 **webapps/ROOT** 이다.

다음과 같이 index2.jsp를 작성해둔다.

```jsp
<html>
<head>
<title>Tomcat Server</title>
</head><body>
<div style="width: 100%; font-size: 80px; font-weight: bold; text-align; center;">
<START OF JAVA CODES>
<%
    out.println("Hello World!");
    out.println("<BR>This is a first JSP Application");
%>
<END OF JAVA CODES>
</div></body></html>
```



톰캣을 편하게 시작/종료하기 위해 서비스로 등록한다.

```bash
$ vim /etc/systemd/system/tomcat.service
------------------------------------------------------------------------------
[Unit]
Description=Apache Tomcat 8
After=syslog.target network.target

[Service]
Type=forking

Environment="JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.302.b08-0.el8_4.x86_64/"
Environment="CATALINA_HOME=/usr/local/tomcat8"

User=root
Group=root

ExecStart=/bin/sh /usr/local/tomcat8/bin/startup.sh
ExecStop=/bin/sh /usr/local/tomcat8/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
------------------------------------------------------------------------------
$ systemctl daemon-reload
```



### 3) mod_jk 설치

192.168.111.100 에서 진행한다.

```bash
$ setsebool -P httpd_can_network_connect 1
$ dnf -y install gcc gcc-c++ httpd-devel
$ wget -c http://mirror.navercorp.com/apache/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.48-src.tar.gz
```

압축풀고 native로 이동한다. ```./configure --with-apxs=/usr/bin/apxs``` 후 ```make && make install``` 로 모듈을 설치한다.

 **/etc/httpd/modules/mod_jk.so** 가 설치되었는지 확인한다.



### 4) Apache 와 Tomcat 연동

```bash
$ vim /usr/local/tomcat8/conf/server.xml
-------------------------------------------------------------------------------
# 117행 주석을 해제한다.
<Connector protocol="AJP/1.3"
           address="0.0.0.0"
           port="8009"
           redirectPort="8443" />
-------------------------------------------------------------------------------
$ systemctl stop tomcat
$ systemctl start tomcat
```



```bash
$ vim /etc/httpd/conf.d/mod_jk.conf
-------------------------------------------------------------------------------
LoadModule jk_module "/etc/httpd/modules/mod_jk.so"
<IfModule mod_jk.c>
	# 파일들의 위치 지정
    JkWorkerFile /etc/httpd/conf/workers.properties
    JkShmFile /var/run/httpd/mod_jk.shm
    JkLogFile /var/log/httpd/mod_jk.log
    # 로그레벨과 시간포맷 지정
    JkLogLevel info
    JkLogStampFormat "[%y-%m-%d %H:%M:%S]"
    
    # jsp, json, xml, do를 가진 경로는 worker로 연결
    JkMount /*.jsp app1Worker
    JkMount /*.json app1Worker
    JkMount /*.xml app1Worker
    JkMount /*.do app1Worker
</IfModule>
```



```bash
$ vim /etc/httpd/conf/workers.properties
------------------------------------------------------------------------------
orkers.apache_log=/var/log/httpd
workers.list=app1Worker
workers.stat1.type=status
workers.app1Worker.type=ajp13
workers.app1Worker.host=192.168.111.200
workers.app1Worker.port=8009
```



```bash
$ setsebool -P domain_can_mmap_files 1
```

