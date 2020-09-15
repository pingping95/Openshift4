# Apache+Tomcat+MariaDB

- References

    [tomcat mysql jdbc 연동하기](https://sangmin-kim.tistory.com/83)

    [WEB ＋WAS + MySQL 연동 설정 :: 2cpu, 지름이 시작되는 곳!](http://2cpu.co.kr/PDS/13141)

    [3-Tier 구성 방법](https://www.teamofmajor.com/tutorial/cloud/3Tier/)

    [6. [CentOS7] 아파치와 톰캣 연동하기 (mod_jk) & 로드밸런스 설정](https://goddaehee.tistory.com/77)

    [리눅스 아파치, 톰캣 연동 및 설치 상세 가이드](https://thereclub.tistory.com/44)

    [아파치 로드밸런싱으로 여러 WAS 운영하기](https://taetaetae.github.io/2019/08/04/apache-load-balancing/)

    [07 CentOS - APM Setup - Tomcat](https://forcloud.tistory.com/53)

    [(AWS) Apache(Web Server)와 Tomcat 연동하기](http://progtrend.blogspot.com/2018/06/aws-apacheweb-server-tomcat.html)

    [JVM 이란?](https://medium.com/@lazysoul/jvm-%EC%9D%B4%EB%9E%80-c142b01571f2)

    [자바 메모리 관리 - 스택 & 힙](https://yaboong.github.io/java/2018/05/26/java-memory-management/)

### 추가 진행해야 할 부분

- Tomcat Load Balancing
- PHP 설치 및 연동

## 실습 환경

selinux는 모두 꺼두고 진행하였습니다.

### MariaDB Server

- IP Address : 192.168.202.12
- Port : 3306/tcp
- Service : MariaDB 5.5.65

MariaDB는 편의상 기본 yum repository를 통해 다운로드 받음

### Tomcat Server

- IP Address : 192.168.202.11
- Port
    - 8080/tcp → ( Direct Tomcat으로 들어오는 Request 전용, Test용이며 Apache와 연동이 완료되면 다시 닫아도 무방 )
    - 8009/tcp → mod_jk Connector를 통해 들어오는 Request이다. 다시 8443으로Redirect됨
    - 8443/tcp
- Service, Installed
    - java-1.8.0-openjdk
    - tomcat
    - tomcat-admin-webapps
    - tomcat-webapps
    - tomcat-docs-webapp

    Tomcat 또한 편의상 yum을 통해 설치하였음

### Apache Server

- IP Address : 192.168.202.10
- Port
    - 80/tcp
    - 443/tcp
    - 8009/tcp
- Service, Installed
    - httpd
    - httpd-devel
    - gcc
    - gcc-c++
    - mod_jk

# 1. Apache - Tomcat 연동

## ################# Apache #################

### Apache 관련 Packege 설치

```jsx
yum -y install httpd httpd-devel
```

- Tomcat과의 연동을 하지 않을 계획이라면 httpd만 따로 설치해도 되지만 Tomcat과 연동할 계획이라면 httpd-devel를 설치해주어야 한다.
- httpd-devel 패키지는 Apache용 모듈을 빌드할 때 사용될 헤더파일들과 도구(apxs)를 설치해준다.
- mod_jk 모듈을 빌드해주기 위해서 반드시 필요하다.

### gcc 설치

```jsx
yum -y install gcc gcc-c++
```

### Tomcat-Connector 설치 (Mod-jk)

[The Apache Tomcat Connectors: mod_jk, ISAPI redirector, NSAPI redirector](http://tomcat.apache.org/connectors-doc/)

해당 사이트에서 Mod-JK-1.2.48 released를 설치받을 수 있다.

그 외, workers.properties, uriworkermap.properties, Apache HTTP Server, AJPv13, LoadBalancing, Reverse Proxy 등에 대한 레퍼런스를 볼 수 있다.

tar.gz 파일의 링크 주소를 복사

![Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled.png](Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled.png)

```bash
>>>> wget이 없다면 설치 
yum -y install wget

>>>> wget을 통하여 설치
wget http://apache.tt.co.kr/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.48-src.tar.gz

>>>> tar 명령어를 통해 압축 풀기
tar zxvf tomcat-connectors-1.2.48-src.tar.gz

>>>> apxs 위치 확인 후 src 폴더 내의 native 폴더로 이동하여 빌드 및 설치
which apxs
/usr/bin/apxs
cd tomcat-connectors-1.2.48-src/native/
./configure --with-apxs=/usr/bin/apxs
make; make install
```

위 과정을 별 문제없이 했다면 mod_jk 모듈의 설치는 완료

### Apaceh-Tomcat 설정 수정

- /etc/http/conf/httpd.conf

vim httpd.conf → :se nu 후에 들어가서 대략 95번째 라인의 ServerName에 있는 초기값 (www.example.com)을Apache Server IP Address:80 로 수정해준다.

만약 변경해주지 않으면 오류가 나며 systemctl status httpd -l을 통해 확인했을 때 이부분에서 오류가 난다는 표시가 뜬다.

위에는 주석에 설명이 잘 되어있다.

![Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%201.png](Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%201.png)

- conf.d 디렉터리에 mod_jk.conf, uriworkermap.properties, [workers.properties](http://workers.properties) 3가지 파일을 추가시켜준다.

```bash
[root@web httpd]# tree
.
├── conf
│   ├── httpd.conf
│   └── magic
├── conf.d
│   ├── autoindex.conf
│   ├── mod_jk.conf
│   ├── README
│   ├── uriworkermap.properties
│   ├── userdir.conf
│   ├── welcome.conf
│   └── workers.properties
├── conf.modules.d
│   ├── 00-base.conf
│   ├── 00-dav.conf
│   ├── 00-lua.conf
│   ├── 00-mpm.conf
│   ├── 00-proxy.conf
│   ├── 00-systemd.conf
│   └── 01-cgi.conf
├── logs -> ../../var/log/httpd
├── modules -> ../../usr/lib64/httpd/modules
└── run -> /run/httpd
```

- mod_jk.conf 생성 후 설정

    아래와 같이 설정해주자.

```bash
[root@web httpd]# cd conf.d
[root@web conf.d]# ls
autoindex.conf  mod_jk.conf  README  uriworkermap.properties  userdir.conf  welcome.conf  workers.properties
[root@web conf.d]# vim mod_jk.conf

[root@web conf.d]# cat mod_jk.conf

LoadModule    jk_module  modules/mod_jk.so

JkWorkersFile conf.d/workers.properties
JkShmFile     run/mod_jk.shm
JkLogFile     logs/mod_jk.log
JkLogLevel    info
JKMountFile conf.d/uriworkermap.properties
JKLogStampFormat "[%a %b %d %H:%M:%s %Y]"
JKRequestLogFormat "%w %V %T"
```

- [workers.properties](http://workers.properties) 생성 후 설정

```bash
[root@web conf.d]# vim workers.properties

[root@web conf.d]# cat workers.properties
########## workers.properties ###########
worker.list=worker1
worker.worker1.type=ajp13
worker.worker1.host=192.168.202.11
worker.worker1.port=8009
```

- [uriworkermap.properties](http://uriworkermap.properties) 생성 후 설정

mod_jk.conf에서 uriworkermap의 설정파일은 conf.d 디렉터리 내에 있다고 설정했다.

```bash
[root@web conf.d]# vim uriworkermap.properties
############## uriworkermap.properties #############

[root@web conf.d]# cat uriworkermap.properties
/*=worker1
```

- /etc/httpd/conf/httpd.conf

    맨 아래에 다음과 같은 내용 추가

    uriworkermap.properties에서 /* (모든 파일)을 worker을 통해 tomcat에 전달하라고 설정을 해주었는데

    apache는 정적인 컨텐츠를 처리하기 위해 존재하는 서버이므로 *.html, *.jpg, *.gif는 apache단에서 처리하도록 준다.

![Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%202.png](Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%202.png)

- apachectl configtest → 문법 체크

apachectl configtest

### 방화벽 오픈 및 데몬 재시작

80, 8009 Port Open

```bash
# firewall-cmd --permanent --zone=public --add-port=80/tcp
success
# firewall-cmd --permanent --zone=public --add-port=8009/tcp
success
# firewall-cmd --reload
success

systemctl restart httpd && systemctl enable httpd
```

```bash
[root@web conf]# cd /var/www/html/
[root@web html]# ls
test.html
[root@web html]# cat test.html
test
```

![Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%203.png](Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%203.png)

- 위의 출력물을 통해 html 확장자를 가진 파일은 web server단에서 처리

## ############ Tomcat Server ###############

### Open-JDK 설치 및 설정

- 무료 오픈소스 openjdk 설치

yum install java-1.8.0-openjdk

- 버전 확인

java -version

### JAVA 환경변수 설정

- 심볼릭 링크 원본 확인

readlink -f /usr/bin/java /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.252.b09-2.el7_8.x86_64/jre/bin/java

- vim /etc/profile

맨 아래로 이동

####JAVA1.8####
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.252.b09-2.el7_8.x86_64
PATH=$PATH:$JAVA_HOME/bin
CLASSPATH=$JAVA_HOME/jre/lib:$JAVA_HOME/lib/tools.jar

export JAVA_HOME PAATH CLASSPATH

source /etc/profile

- 환경변수 확인

echo $JAVA_HOME

echo $PATH

echo $CLASSPATH

### Tomcat 설치

- yum -y install tomcat

설치 후 테스트 및 간단한 서버 관리를 위해 아래 패키지들을 설치

- yum -y install tomcat-webapps
- yum -y install tomcat-admin-webapps
- yum -y install tomcat-docs-webapp

설치 완료 후 Tomcat의 모든 설정파일은 /etc/tomcat 폴더에 들어있다. Tomcat 설정할 때 가장 많이 건드리는 파일이 server.xml이다.

HTTP/1.1 요청은 8080

AJP/1.3 요청은 8009로 받도록 되어있다.

### Tomcat 데몬 시작, 재부팅 후에도 활성화 설정

- systemctl restart tomcat
- systemctl enable tomcat

### server.xml 설정파일 수정

- vim /etc/tomcat/server.xml

address, secretRequired, URIEncoding 추가

![Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%204.png](Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%204.png)

```bash
<!-- Define an AJP 1.3 Connector on port 8009 -->
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" address="0.0.0.0" \
secretRequired="false" URIEncoding="UTF-8" />
```

### 방화벽 오픈 및 tomcat 데몬 재시작

firewall-cmd --permanent --zone=public --add-port=8009/tcp

firewall-cmd --permanent --zone=public --add-port=8080/tcp

firewall-cmd --permanent --zone=public --add-port=8443/tcp

firewall-cmd --reload

firewall-cmd --list-all

systemctl restart tomcat && systemctl enable tomcat

### Apache → Tomcat

![Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%205.png](Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%205.png)

- Apache의 IP:80 을 통해 톰캣 서버로 접속된것을 확인할 수 있다.

# 2. tomcat - MariaDB 연동

## MySQL

1. mariadb-server mariadb 설치

2. 3306 port open

```bash
yum -y install mariadb-server mariadb

systemctl start mariadb && systemctl enable mariadb

firewall-cmd --permanent --add-port=3306/tcp

firewall-cmd --reload

firewall-cmd --list-all
```

3. mysql 설정

```bash
MariaDB [(none)]> CREATE DATABASE testdb default character set utf8;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON testdb.* to 'testuser'@'localhost' identified b
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON testdb.* to 'testuser'@'%' identified by '1234'
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> bye
    -> ;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that correspon MariaDB server version for the right syntax to use near 'bye' at line 1
MariaDB [(none)]> exit
Bye
[root@db ~]# mysql -u testuser -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 11
Server version: 5.5.65-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| testdb             |
+--------------------+
2 rows in set (0.00 sec)

MariaDB [(none)]> USE testdb;
Database changed
MariaDB [testdb]> show tables;
Empty set (0.00 sec)

MariaDB [testdb]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| testdb             |
+--------------------+
2 rows in set (0.00 sec)

MariaDB [testdb]> create table testtb(name varchar(30));
Query OK, 0 rows affected (0.01 sec)

MariaDB [testdb]> select * from testtb;
Empty set (0.00 sec)

MariaDB [testdb]> insert into test values('MariaDB db');
ERROR 1146 (42S02): Table 'testdb.test' doesn't exist
MariaDB [testdb]> insert into testtb values('MariaDB db');
Query OK, 1 row affected (0.01 sec)

MariaDB [testdb]> select * from testtb;
+------------+
| name       |
+------------+
| MariaDB db |
+------------+
1 row in set (0.00 sec)

MariaDB [testdb]> insert into testtb values('taehoon');
Query OK, 1 row affected (0.00 sec)

MariaDB [testdb]> select * from testtb;
+------------+
| name       |
+------------+
| MariaDB db |
| taehoon    |
+------------+
2 rows in set (0.00 sec)

MariaDB [testdb]> commit
    -> ;
Query OK, 0 rows affected (0.00 sec)

MariaDB [testdb]> quit
Bye
```

## Tomcat Server

- JDBC를 통하여 DB에 연결하기 위해서는 드라이버(Driver)를 Load하고 커넥션(Connection) 객체를 받아와야 한다.
- 커넥션풀은 웹 컨테이너가 실행되면서 커넥션 객체를 미리 Pool에 생성해 둔다.
- DB와 연결된 커넥션을 미리 생성해서 풀(Pool)속에 저장해 두고 있다가 필요할 때에 가져다 쓰고 반환한다.
- 미리 생성해두기 때문에 데이터베이스의 부하를 줄이고 유동적으로 관리할 수 있어서 Connection Pool을 사용한다.

1. JDBC Driver for MYSQL (Connector/J) 다운로드

```bash
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.48.tar.gz

## tar 압축 해제
tar xzf mysql-connector-java-5.1.48.tar.gz
```

2. 해당 connector 디럭터리에 JAVA, TOMCAT 설치 경로에 복사

```bash
# JAVA 설치 경로 복사
cp mysql-connector-java-5.1.48-bin.jar /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.222.b10-0.47.amzn1.x86_64/jre/lib/ext/

# Tomcat 설치 경로 복사
cp mysql-connector-java-5.1.48-bin.jar /usr/share/tomcat7/lib/
```

3. context.xml 파일 설정

→ context 내부에 추가해준다.

![Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%206.png](Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%206.png)

```bash
<Resource       name="jdbc/dbmy"
                auth="Container"
                type="javax.sql.DataSource"
                maxActive="50"
                maxIdle="30"
                maxWait="10000"
                username="testuser"
                password="1234"
                driverClassName="com.mysql.jdbc.Driver"
                url="jdbc:mysql://192.168.202.12:3306/testdb"/>
</Context>

# jdbc의 이름은 dbmy이다.
# username은 사전에 testdb에서 생성한 user의 이름을 적어준다.
# password는 testuser의 password를 적어준다.
# maxActive는 최대로 연결을 허용하는 갯수이다.
# maxIdle은 항상 연결상태를 유지하는 개수이다.
# url은 IP_주소:3306(DB Port)/testdb(Database Name)을 적어준다. IP 대신 Domain Name을 적어줘도 무방
```

4. web.xml 파일 설정

→ 톰캣의 실행환경에 대한 정보를 담당하는 환경설정 파일이다.

→ 각종 servlet의 설정과 servlet 맵핑, 필터, 인코딩 등을 담당한다.

→ web.xml은 톰캣에 있는 모든 web application의 기본설정을 정의한다.

→ context에 적어준 resource명을 가져옴으로써 사용한다.

![Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%207.png](Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%207.png)

```bash
<resource-ref>
<description>MariaDB Datasource</description>
<res-ref-name>jdbc/dbmy</res-ref-name>
<res-type>javax.sql.DataSource</res-type>
<res-auth>Container</res-auth>
</resource-ref>
```

5. DB 연동 테스트를 위한 mariadb_test.jsp 파일을 생성

![Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%208.png](Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%208.png)

```bash
cd /var/lib/tomcat7/webapps/ROOT

<%@ page language="java" contentType="text/html; charset=UTF-8"
       pageEncoding="UTF-8" import="java.sql.*"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<title>DB Connection Test</title>
</head>
<body>
       <%
              String DB_URL = "jdbc:mysql://192.168.202.12:3306/"testdb";
              String DB_USER = "testuser";
              String DB_PASSWORD = "1234";
              Connection conn;
              Statement stmt;
              PreparedStatement ps;
              ResultSet rs;
              try {
                     Class.forName("com.mysql.jdbc.Driver");
                     conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
                     stmt = conn.createStatement();

                    /* SQL 처리 코드 추가 부분 */

                     conn.close();
                     out.println("MySQL JDBC Driver Connection Test Success!!!");

/* 예외 처리  */
              } catch (Exception e) {
                     out.println(e.getMessage());
              }
       %>
</body>
</html>
```

- Test ⇒ apache server_ip/mariadb_test.jsp

![Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%209.png](Apache+Tomcat+MariaDB%20c9e97d58812e4bd699ae33209e566a94/Untitled%209.png)

### Apache → Tomcat → MariaDB

- apache ip를 통해 db에서 data_resource를 받아와 client에게 출력


