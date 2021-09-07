---
title: "[CentOS 8] 아파치 (Apache) 서버"
date: '2021-09-01 00:26:14'
categories: Linux
toc: true
toc_sticky: true
sidebar:
  nav: docs
---
## 1. 아파치 주요 디렉토리

리눅스에서 아파치 웹 서버는 httpd 를 통해 운용할 수 있다. 

```dnf -y install httpd``` 로 설치 후 데몬을 시작하고 방화벽을 열어준다.

### 주요 디렉토리

**/usr/sbin/httpd** : 아파치 데몬 위치

**/var/www/html** : 웹 문서들이 들어가는 기본 경로

**/var/www/cgi-bin** : cgi 파일들의 기본 경로

**/etc/httpd/conf** : 아파치 주 설청파일의 경로

**/etc/httpd/conf.d** : 아파치 추가 설정파일의 경로

**/var/logs/httpd** : 웹 로그파일의 경로 (access_log, error_log)

**/usr/share/httpd/error** : 아파치 에러코드에 따른 에러문서들의 경로

**/usr/lib64/httpd/modules** : DSO(Dynamic Shared Object) 방식의 아파치에서 로드할 모듈파일의 경로



## 2. 아파치 설정파일

아파치 설정파일은 **/etc/httpd/conf/httpd.conf** 이다.



**ServerRoot** : 아파치의 홈 디렉토리를 지정한다. 이후에 나오는 경로들은 이 경로를 루트로 한 상대경로로 지정된다.

**Listen** : 시스템 기본값 이외에 바인드할 포트를 지정한다.

**Include** : DSO를 위해 사용할 모듈을 적재한다.

**ServerAdmin** : 웹 에러페이지에 보여질 관리자 메일주소이다.

**ServerName** : 클라이언트에게 보여주는 호스트 이름이다. 현재 사용되는 도메인이 없다면 IP 주소를 적어야 한다.

**UseCanonicalName** : On 일 경우, 아파치가 자기 참조 URL을 만들 필요가 있을 때마다 공식적인 이름을 만들기 위해 ServerName과 Port를 사용한다. Off 일 경우, 아파치는 클라이언트가 제공한 HostName과 Port를 사용한다.

**DocumentRoot** : 서버의 웹 문서가 있는 경로이다. 심볼릭 링크나 Alias를 사용해 다른 위치를 가리키도록 할 수 있다.

**DirectoryIndex** : 웹사이트의 초기 페이지 문서로 보여줄 파일이다. 적혀진 순서에 따라 읽는다.

**AccessFileName** : 각 디렉토리에 대하여 접근 제어 정보를 담고 있는 파일이름을 설정한다. 통상적으로 .htaccess 이다.

**ErrorLog** : 에러 로그파일의 경로이다.

**Alias <별칭명> <실제명>** : Alias 기능이다.

**AddHandler** : 파일 확장자를 처리기에 연결되게 해준다. 만약 .php 를 적는다면 *.php 형식의 파일들은 php5-scripter라는 handler가 처리하라고 아파치 웹 서버에게 알려준다.



### Directory 지시자

**<Directory /> </Dirctory>**: Directory 지시자는 지정한 디렉토리 이하의 모든 웹문서들에 대하여 어떤 서비스와 기능을 허용/거부할 것인지를 설정하는 지시자이다. 다른 지시자들을 포함한다.

**Options** : 지정한 디렉토리 이하의 모든 파일과 디렉토리들에 적용할 접근제어를 설정한다. 

뒤에 다음과 같은 옵션이 붙는다.

|      옵션      |                             설명                             |
| :------------: | :----------------------------------------------------------: |
|      None      | 모든 허용을 하지 않는다. 즉, 다른 설정값들을 모두 무시한다.  |
|      All       | MultiViews를 제외한 모든 옵션설정을 허용한다. Options 값을 공백으로 두어도 All로 설정된다. |
|     Indexs     | 웹서버의 디렉토리 접근시에 DirctoryIndex에서 지정한 파일이 존재하지 않을 경우에 디렉토리 내 파일리스트를 웹브라우저로 보여준다. |
|    Includes    |                     SSI 사용을 허용한다.                     |
| IncludesNOEXEC | SSI 사용은 허용되지만 #exec 사용과 #include는 허용되지 않는다. |
| FollowSymlinks |                    심볼릭 링크를 허용한다                    |
|    ExecCGI     |               perl과 같은 CGI 실행을 허용한다                |
|   MultiViews   | 웹브라우저의 요청에 따라 적절한 페이지로 보여준다. 웹브라우저의 종류나 웹문서의 종류에 따라 가장 적합한 페이지를 보여줄 수 있도록 하는 설정이다. |

상위 디렉토리의 다른 옵션은 변경하지 않고 특정 옵션만 제거하거나 추가할 때 옵션 앞에 + 나 - 를 붙여 사용한다.



**AllowOverride** : 어떻게 접근을 허락할 것인가에 대한 설정이다. 특정 디렉토리에 대한 방문자들의 접근방식을 어떤 방식으로 인증하여 허용할 것인가의 문제라고 할 수 있다. AllowOverride 에서 설정하는 값들은 중복해서 설정될 수 있으며 그때마다 가장 최근에 설정된 값이 우선적용된다.

뒤에 다음과 같은 옵션이 붙는다.

|    옵션    |                             설명                             |
| :--------: | :----------------------------------------------------------: |
|    None    | AccessFileName에 지정된 파일을 엑세스 인증파일로 인식하지 않는다. 즉 .htaccess 파일을 무시하여 아주 제한적인 접근만을 허용할 때 사용된다. |
|    All     | 이전의 인증방식에 대하여 새로운 접근인증방식을 우선적죵하도록 Override를 허용한다 |
| AuthConfig | AccessFileName 지시자에 명시한 파일에 대하여 클라이언트 인증지시자의 사용을 허용한다. 특정 디렉토리 접근을 AccessFileNAme에 명시한 파일로 제어하고자 할 때 사용된다 |
|  FileInfo  | AccessFileName 지시자에 명시한 파일에 대하여 문서유형을 제어하는 지시자 사용을 허용한다. |
|  Indexes   | AccessFileName 지시자에 명시한 파일에 대하여 디렉토리 인덱싱을 제어하는 지시자 사용을 허용한다. |
|  Options   | AccessFileName 지시자에 명시한 파일에 대하여 디렉토리 옵션을 제어하는 지시자 사용을 허용한다. |
|   Limit    | AccessFileName 지시자에 명시한 파일에 대하여 호스트 접근을 제어하는 지시자 사용을 허용한다. |



**Order** : 접근을 통제하는 순서이다. allow deny 순으로 지정하면 allow기능을 먼저 수행하고 deny 기능을 나중에 수행한다. mutual-failure는 allow 목록에 없는 모든 호스트 접근을 차단한다. 