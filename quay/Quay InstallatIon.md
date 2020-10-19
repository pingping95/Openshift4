# Quay Installation

# Quay 란?

- 기업 환경을 위한 분산 고가용성 컨테이너 이미지 레지스트리이다.
- On-Premises, Public Cloud, Hosted-Service로 제공한다.
- 모든 컨테이너 환경 혹은 오케스트레이션 환경에서 작동한다.
- 주요 특징
    - 확장성 - 대규모 스케일 Test
    - 빌드 자동화 - Git과 완벽한 통합
    - 통합 - 확장 가능한 API
    - 보안
    - 컨텐츠 분산
    - 접근 제어

## 구현할 Architecture

- Quay가 외부 Source Registry로부터 Image를 받아와 Mirroring한 뒤에 Clair을 통해 CVE Metadata를 Fetch하고 Image 취약점을 검사한다.
- Quay는 Openshift Container Platform의 외부 Node로써 구현된다.

![Untitled](https://user-images.githubusercontent.com/67780144/96406775-19ba7880-121b-11eb-81af-3558770a43b2.png)

- Quay Node 내부 컨테이너들간의 관계

![Untitled 1](https://user-images.githubusercontent.com/67780144/96406779-1aeba580-121b-11eb-8414-094c840ef46d.png)

- Database : Quay에서 기본 Metadata Storage로 사용
- Redis (Key, Value Store) : Live Builder Log 및 Red Hat Quay turorial 저장
- Quay (Container Registry) : Quay Container를 Pod의 여러 구성 요소로 구성된 서비스로 실행
- Clair : Container 이미지들의 취약점을 검사하고 수정 사항을 제안
- Object Storage : 오브젝트 형태의 데이터를 저장할 수 있는 스토리지 또한 필요
    - On-Premises에서는 Ceph Rados RGW, Openstack Swift, NooBaa, Minio 등이 있다.
    - Public Cloud에서는 S3, Google Cloud Storage, Azure Blob Storage 등이 있다.

## Quay Node - Containers 간의 Port Forwarding

- Quay
    - QuayBuilder : 80 Port
    - Redis : 6379 Port
    - MySQL : 3306 Port
    - QuayConfig : 28443 Port
    - Minio : 9000 Port
    - Gitlab : 18443, 18080, 18022 Port
- Clair
    - JWTproxy : 6060 Port
    - pgsql : 5432 Port
    - Postgresql : pgsql과 Link 연결

# Red Hat Quay Installation

- quay : 3.2.1 버전
- quay-builder : 3.2.1 버전
- clair-jwt : 3.2.1 버전

## Environment

- RHEL 7.8
- RAM : 8 GB ( 4GB로 진행하다가 중간에 Resource 부족 현상으로 인해 8GB로 늘려줌)
- vCPU : 3 Core
- Disk : 60 GB
- NIC
    - External IP : 192.168.100.10/24 (NAT)
    - Internal IP : 10.10.10.21
- Firewall은 해제하고 진행함
- Hostname : quay.redhat2.cccr.local

## Prerequisites

- RHEL Subscription

## 초기 Settings

- hostname 설정

```bash
# Quay Node의 Hostname 변경
hostnamectl set-hostname quay.redhat2.cccr.local

# Quay Node에서 추가
# /etc/hosts
192.168.100.10	quay.redhat2.cccr.local
192.168.100.10	clair.redhat2.cccr.local
192.168.100.10	gitlab.redhat2.cccr.local

# Host PC에서 추가 (나중에 웹 콘솔로 접속하기 위함)
# /etc/hosts
192.168.100.10	quay.redhat2.cccr.local
192.168.100.10	clair.redhat2.cccr.local
192.168.100.10	gitlab.redhat2.cccr.local
```

- Subscription 등록 및 필요한 Repository 활성화

```bash
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

# 본인의 Redhat 계정을 기입한다.
subscription-manager register
Username: ${USERNAME}
Password: ${PASSWORD}

The system has been registered with ID: ~~~~
The registered system name is: quay.redhat2.cccr.local

subscription-manager list --available --matches '*RHEL*'

subscription-manager attach --pool=<<Pool_ID>>

subscription-manager repos --disable="*"

subscription-manager repos \
--enable="rhel-7-server-rpms" \
--enable="rhel-7-server-extras-rpms"

yum update -y

# 업데이트 끝나면 Server Reboot 해주기
reboot
```

## Docker 설치 및 설정, quay 인증

- Docker Install

```bash
yum install docker device-mapper-libs device-mapper-event-libs**

systemctl enable docker

systemctl start docker

systemctl is-active docker
```

- Insecure Docker Registry 세팅

    /etc/sysconfig/docker 파일 하단에 추가

```bash
# vi /etc/sysconfig/docker

ADD_REGISTRY='--add-registry quay.redhat2.cccr.local'
INSECURE_REGISTRY='--insecure-registry quay.redhat2.cccr.local'
```

- [quay.io](http://quay.io) 인증

    [quay.io](http://quay.io) (quay.io/redhat) 에서 Red Hat Quay V3 Container Image를 얻으려면 아래의 Key를 사용하여야 한다. ( ## 처리함 )

```bash
[root@quay ~]# docker login -u="<<ID>>" \
-p="<<PASSWORD>>" quay.io
```

## Quay Install

### 1. Database Install & Deploy

- PostgreSQL 혹은 MySQL을 선택해준다.
- 해당 메뉴얼은 MySQL Database Container를 배포한다.

```bash
# 디렉터리 생성
mkdir -p /var/lib/mysql

chmod -R 777 /var/lib/mysql

# 환경변수 설정
export MYSQL_CONTAINER_NAME=mysql
export MYSQL_DATABASE=quay
export MYSQL_PASSWORD=quay
export MYSQL_USER=quay
export MYSQL_ROOT_PASSWORD=quay

# MySQL Container 설치 및 배포
docker run \
--detach \
--restart=always \
--env MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD} \
--env MYSQL_USER=${MYSQL_USER} \
--env MYSQL_PASSWORD=${MYSQL_PASSWORD} \
--env MYSQL_DATABASE=${MYSQL_DATABASE} \
--name ${MYSQL_CONTAINER_NAME} \
--publish 3306:3306 \
-v /var/lib/mysql:/var/lib/mysql/data:Z \
registry.access.redhat.com/rhscl/mysql-57-rhel7
```

- Test

```bash
# MySQL 사용자 계정 패스워드를 정적으로 설정하지 않고 생성하려면 다음을 실행
export MYSQL_PASSWORD=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | sed 1q)

export MYSQL_ROOT_PASSWORD=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | sed 1q)

# Check
yum install -y mariadb
mysql -h 192.168.100.10 -u root --password=quay

# 실행 결과

[root@quay ~]# mysql -h 192.168.100.10 -u root --password=quay
Welcome to the MariaDB monitor.
Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.24 MySQL Community Server (GPL)
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
Type 'help;' or '\h' for help. Type '\c' to clear the current input
statement.
MySQL [(none)]> \q
Bye
```

### 2. Redis Install & Deploy

- Redis 설치 및 배포

```bash
# Redis 디렉터리 생성

mkdir -p /var/lib/redis

chmod -R 777 /var/lib/redis

docker run -d --restart=always -p 6379:6379 \
--privileged=true \
-v /var/lib/redis:/var/lib/redis/data:Z \
registry.access.redhat.com/rhscl/redis-32-rhel7
```

- Test

```bash
yum install telnet -y

telnet 192.168.100.10 6379

[root@quay ~]# telnet 192.168.100.10 6379
..
..
Escape character is '^]'.
MONITOR
+OK
QUIT
+OK
Connection closed by foreign host.
```

### 3. Gitlab Install

- Docker Container를 통해 간편하게 Gitlab을 설치할 수 있다.

```bash
# Gitlab Directory 생성
# config, logs, data

mkdir -p /srv/gitlab/config
mkdir -p /srv/gitlab/logs
mkdir -p /srv/gitlab/data

chmod -R 777 /srv/gitlab

chown 1000:1000 /srv/gitlab

# Context 변경
# /srv 디렉터리로 가서 실행
chcon -Rv -u system_u *
chcon -Rv -t container_file_t *

# 아래와 같은 출력물이 나와야 한다.
[root@quay1 srv]# ls -lZ *
drwxrwxr-x. root root system_u:object_r:container_file_t:s0 config
drwxr-xr-x. root root system_u:object_r:container_file_t:s0 data
drwxr-xr-x. root root system_u:object_r:container_file_t:s0 logs
```

- Gitlab 설치 및 실행

```bash
docker run --detach \
--hostname gitlab.redhat2.cccr.local \
--publish 18443:443 --publish 18080:80 --publish 18022:22 \
--name gitlab \
--restart always \
--volume /srv/gitlab/config:/etc/gitlab \
--volume /srv/gitlab/logs:/var/log/gitlab \
--volume /srv/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce:latest
```

### 4. Minio 설치 및 배포

```bash
# Minio 디렉터리 생성
mkdir /mnt/minio
chmod -R 777 /mnt/minio

# Minio 생성 및 배포
docker run -itd -p 9000:9000 \
--name minio1 --privileged=true \
-e "MINIO_ACCESS_KEY=minioadmin" \
-e "MINIO_SECRET_KEY=minioadmin" \
-v /mnt/minio:/data minio/minio server /data
```

- Minio 접속

    접속 후 quay 디렉터리를 생성해주었다.

![Untitled 2](https://user-images.githubusercontent.com/67780144/96406783-1d4dff80-121b-11eb-89b0-220abde5e502.png)

### 5. quayconfig를 통한 Red Hat Quay 설정

- Container로 Red Hat Quay를 사용하기 전에 동일한 Quay Container를 사용하여 Red Hat Quay를 사용하는데 필요한 구성 파일 (config.yaml)과 security_scanner.pem를 생성해주어야 한다.
- 이는 Quay가 어떤 Redis, MySQL, Gitlab, Clair와 연동할 것인지, Quay Web Console 접속할 시 사용할 ID, Password 등에 대해 정의해주는 config.yaml이다.

```bash
docker run --privileged=true -p 28443:8443 \
-d quay.io/redhat/quay:v3.2.1 config password
```

- Docker Networking Disabled: WARNING: IPv4 forwarding is disabled. Networking will not work 메시지 발생 할 경우 (무조건 해주어야 하는 것은 아닙니다)

    ⇒ Docker Process가 ipv6로 떠 있는 것을 확인 할 수 있고, ipv4로 띄우기 위해 다음 설정이 필
    요함

    ```bash
    # ipv4 forwarding 활성화 ( /etc/sysctl.conf 수정 )
    net.ipv4.ip_forward = 1

    systemctl restart network
    ```

- Host PC의 Web 브라우저로 quay 구성 도구가 실행중인 URL 접속

```bash
# 해당 URL로 접속하면 quayconfig가 뜰 것임
https://quay.redhat2.cccr.local:28443/
```

### quayconfig

- DB 정보 입력

![Untitled 3](https://user-images.githubusercontent.com/67780144/96406785-1de69600-121b-11eb-99d9-1596884339bf.png)

- Super User 생성

    Super User 생성을 위한 username, Email Address, Password, Repeat Password를 생성

![Untitled 4](https://user-images.githubusercontent.com/67780144/96406787-1de69600-121b-11eb-9399-2c2014822711.png)

- Server Configuration

    Server Hostname : [quay.redhat2.cccr.local](http://quay.redhat2.cccr.local) 기입

- Time Machine

    Default로 놔둠

- Redis

    Redis Hostname : 192.168.100.10

    Redis Port : 6379

- Repository Mirroring
    - Repository Mirroring > Enable Repository Mirroring Check

![Untitled 5](https://user-images.githubusercontent.com/67780144/96406790-1e7f2c80-121b-11eb-8d9c-80f170fcf7e3.png)

- Security Scanner
    - Security Scanner > Enable Security Scanning : Check 해줌
    - Security Scanner Endpoint : [http://clair.redhat2.cccr.local:6060](http://clair.redhat2.cccr.local:6060)

        ⇒ Clair Container의 Endpoint를 기입해준다.

    - Authentication Key 다운로드 받기
        - Assign New Key 링크 클릭!
        - key name : security_scanner Service Key

        ![Untitled 6](https://user-images.githubusercontent.com/67780144/96406791-1e7f2c80-121b-11eb-966f-e5c61e15aefa.png)

        - 다운로드 한 키는 Clair의 Directory에 복사해주면 된다.
        - /var/lib/clair-config (없을 시 생성)
- Application Registry
    - Application Registry >> Enable App Registry 체크!

        ![Untitled 7](https://user-images.githubusercontent.com/67780144/96406793-1f17c300-121b-11eb-9663-e277c32d4bcd.png)

- Registry Protocol Settings
    - Registry Protocol Settings > Restrict V1 Push Support Check Disables

        ![Untitled 8](https://user-images.githubusercontent.com/67780144/96406794-1fb05980-121b-11eb-9e32-d39ad85a44cf.png)

- Minio 설정
    - Bucket Name 같은 경우는 위의 Minio 웹 콘솔로 접속 시 만든 디렉터리 이름을 기입해준다

![Untitled 9](https://user-images.githubusercontent.com/67780144/96406795-2048f000-121b-11eb-84d7-1a5eb3503411.png)

- Gitlab
    - Application ID와 Secret이 필요하므로 Gitlab 웹 콘솔에 접속한다.
    - [http://gitlab.redhat2.cccr.local](http://gitlab.redhat2.cccr.local) 으로 접속

        (안될 시 host PC의 /etc/hosts에 gitlab.redhat2.cccr.local 추가되었는지 확인)

    - New Password : dkag1..
    - ID : root
    - New Password : dkagh1..

    ![Untitled 10](https://user-images.githubusercontent.com/67780144/96406796-2048f000-121b-11eb-9baf-b1673f514751.png)


    - Settings → Application 접근

    ![Untitled 11](https://user-images.githubusercontent.com/67780144/96406799-20e18680-121b-11eb-8594-04c3d25df3bf.png)

    - Name : testapp
    - Redirect URI : [http://gitalb.redhat2.cccr.local/testapp](http://gitalb.redhat2.cccr.local/testapp) (테스트용으로 생성해줌)

    ![Untitled 12](https://user-images.githubusercontent.com/67780144/96406803-217a1d00-121b-11eb-8beb-2b681f279c0f.png)

    - App ID와 Secret이 필요함

    ![Untitled 13](https://user-images.githubusercontent.com/67780144/96406804-2212b380-121b-11eb-86ae-7f1729ee7ad8.png)

    - Enable Gitlab Trigger을 체크하면 아래의 기입란이 나옴
        - Endpoint, App ID, Secret 기입

    ![Untitled 14](https://user-images.githubusercontent.com/67780144/96406806-22ab4a00-121b-11eb-8151-95b0d70079b7.png)

    - 오류 발생하여 gitLab Endpoint를 아래와 같이 ipv4로 수정해주었음

    ![Untitled 15](https://user-images.githubusercontent.com/67780144/96406807-2343e080-121b-11eb-850b-c9b18506a433.png)

    - 설정 정보가 포함된 File Download

    ![Untitled 16](https://user-images.githubusercontent.com/67780144/96406809-23dc7700-121b-11eb-9dbe-be999d8ab1ba.png)

### 6. Quay 설치 및 배포

- docker ps로 컨테이너 상태 확인

    현재까지 minio, quay (설정 컨테이너), gitlab, redis, mysql 를 구축 완료

```bash
[root@quay clair-config]# docker ps
CONTAINER ID        IMAGE                                             COMMAND                  CREATED             STATUS                    PORTS                                                                  NAMES
4720d202d0e3        minio/minio                                       "/usr/bin/docker-e..."   18 minutes ago      Up 18 minutes             0.0.0.0:9000->9000/tcp                                                 minio1
2575c3050175        quay.io/redhat/quay:v3.2.1                        "/quay-registry/qu..."   31 minutes ago      Up 31 minutes             7443/tcp, 8080/tcp, 0.0.0.0:28443->8443/tcp                            kind_pare
7a33ff0ce108        gitlab/gitlab-ce:latest                           "/assets/wrapper"        37 minutes ago      Up 37 minutes (healthy)   0.0.0.0:18022->22/tcp, 0.0.0.0:18080->80/tcp, 0.0.0.0:18443->443/tcp   gitlab
626c3eb9fd77        registry.access.redhat.com/rhscl/redis-32-rhel7   "container-entrypo..."   44 minutes ago      Up 44 minutes             0.0.0.0:6379->6379/tcp                                                 stupefied_dubinsky
5905964d5a75        registry.access.redhat.com/rhscl/mysql-57-rhel7   "container-entrypo..."   46 minutes ago      Up 46 minutes             0.0.0.0:3306->3306/tcp                                                 mysql
```

- 중간에 속도가 매우 느려져 메모리의 상태를 확인해보았더니 꽉 찬것을 확인할 수 있었다.

![Untitled 17](https://user-images.githubusercontent.com/67780144/96406811-24750d80-121b-11eb-88aa-4dc1eca685f7.png)

- 4GB → 8GB 로 늘려줌으로 해결

![Untitled 18](https://user-images.githubusercontent.com/67780144/96406813-250da400-121b-11eb-91d9-c25c6280e52e.png)

- Quay 설정파일 옮기기

```bash
# Quay에서 사용할 디렉토리를 생성
mkdir -p /mnt/quay/config

mkdir -p /mnt/quay/storage

# Host PC에서 Quay VM으로 quay-config.tar.gz 옮김
# 현재 작업민 Host PC에서 진행
scp quay-config.tar.gz root@192.168.100.10:/mnt/quay/config/

# Quay 설정 파일을 /mnt/quay/config/에 업로드 후 압축을 해제한다.
# 이제부터 다시 Quay VM에서 진행
cd /mnt/quay/config

tar xvf quay-config.tar.gz

chcon -Rv -u system_u *.yaml
chcon -Rv -t container_file_t *
```

- quay 설치 및 배포

     - -add-host gitlab.redhat2.cccr.local:192.168.100.10의 경우 builderworker가 quay와 같은 서버에 설치될 경우 no_such_host와 같은 에러가 발생할 수 있으므로 기동되는 docker container에 gitlab host 정보를 추가하여 기동할 수 있다.

```bash
docker run --restart=always -p 443:8443 -p 80:8080 \
--add-host gitlab.redhat2.cccr.local:192.168.100.10 \
--sysctl net.core.somaxconn=4096 \
--privileged=true \
-v /mnt/quay/config:/conf/stack:Z \
-v /mnt/quay/storage:/datastorage:Z \
-d quay.io/redhat/quay:v3.2.1
```

- [quay.redhat2.cccr.local](http://quay.redhat2.cccr.local)

    admin / password 로 접속

![Untitled 19](https://user-images.githubusercontent.com/67780144/96406815-25a63a80-121b-11eb-96d0-c85c39f9f8f9.png)

![Untitled 20](https://user-images.githubusercontent.com/67780144/96406818-263ed100-121b-11eb-84b0-85e1e0221b02.png)

### 7. Mirror Worker 실행

```bash
docker run -d --name mirroring-worker \
-v /mnt/quay/config:/conf/stack:Z \
-d quay.io/redhat/quay:v3.2.1 repomirror
```

### 8. Builder 실행

- ws://quay.redhat2.cccr.local:80 으로 줄 경우 Name Resolving이 안되는 오류가 발생하여 ipv4로 주었음
- 실행 시 환경 변수로 SERVER를 사용하여 worker가 Red Hat Quay에 접근할 수 있는 Host이름을 지정해야 하며, TLS를 사용하지 않는 경우 ws 파라미터와 함께 Port를 명시해주어야 한다.
- TLS를 사용할 경우 wss 파라미터를 사용해주어야 한다.

```bash
docker run --restart on-failure \
-e SERVER=ws://192.168.100.10:80 \
--privileged=true \
-v /var/run/docker.sock:/var/run/docker.sock:Z \
-d quay.io/redhat/quay-builder:v3.2.1
```

## Red Hat Quay와의 연동을 위한 Clair 설치

- OCP 환경에서 Clair 이미지 스캔 컨테이너 및 관련 Database를 실행하려면 Clair를 구성해야 한다.
- PostgreSQL와 Quay를 구축해준다.

### 1. 사전 준비

```bash
# Clair 디렉터리 생성

mkdir -p /var/lib/clair-config
chmod 777 /var/lib/clair-config
cd /var/lib/clair-config

# Docker Image Pull이 안되는 경우 (Optional이므로 필수 사항은 아님)

[root@quay1 clair-config]# docker login docker.io
Login with your Docker ID to push and pull images from Docker Hub. If you
don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: ${USERNAME}
Password: ${PASSWORD}
Login Succeeded
```

### 2. PostgreSQL 설치 및 배포

- Clair를 실행하려면 데이터베이스가 필요하다. Quay에서는 MySQL과 PostgreSQL 모두 가능하지만 Clair에서는 PostgreSQL로만 연동이 가능하다.

```bash
# pgsql Container
docker run -d -p 5432:5432 --name pgsql -e POSTGRES_PASSWORD=mysecretpassword \
-e POSTGRES_HOST_AUTH_METHOD=trust postgres

# postgres Container
docker run --rm --link pgsql:postgres postgres \
sh -c 'echo "create database clairtest" | psql -h \
"$POSTGRES_PORT_5432_TCP_ADDR" -p \
"$POSTGRES_PORT_5432_TCP_PORT" -U postgres'
```

### 3. Clair Image Pull ( Security-enabled )

- Clair 설정 파일 생성
- vi /var/lib/clair-config/config.yaml

```bash
### vi /var/lib/clair-config/config.yaml

clair:
  database:
    type: pgsql
# A PostgreSQL Connection string pointing to the Clair Postgres database.
# Documentation on the format can be found at: http://www.postgresql.org/docs/9.4/static/libpq-connect.html
    options:
      source: postgresql://postgres@192.168.100.10:5432/clairtest? sslmode=disable
      cachesize: 16384
# The port at which Clair will report its health status. For example, if Clair is running at
# https://clair.mycompany.com, the health will be reported at
# http://clair.mycompany.com:6061/health.
  api:
    healthport: 6061
    port: 6062
    timeout: 900s
# paginationkey can be any random set of characters. *Must be the same across all Clair instances*.
    paginationkey:
  updater:
# interval defines how often Clair will check for updates from its upstream vulnerability databases.
    interval: 6h
    enabledupdaters:
    - debian
    - ubuntu
    - rhel
    - oracle
    - alpine
    - suse
    notifier:
      attempts: 3
      renotifyinterval: 1h
      http:
# QUAY_ENDPOINT defines the endpoint at which Quay is running.
# For example: https://myregistry.mycompany.com
# Example: http://quay.redhat2.cccr.local
        endpoint: http://quay.redhat2.cccr.local/secscan/notify
        proxy: http://localhost:6063
jwtproxy:
  signer_proxy:
  enabled: true
  listen_addr: :6063
  ca_key_file: /certificates/mitm.key  # Generated internally, do not change.
  ca_crt_file: /certificates/mitm.crt  # Generated internally, do not change.
  signer:
    issuer: security_scanner
    expiration_time: 5m
    max_skew: 1m
    nonce_length: 32
    private_key:
      type: autogenerated
      options:
        rotate_every: 12h
        key_folder: /clair/config/
        key_server:
          type: keyregistry
          options:
# QUAY_ENDPOINT defines the endpoint at which Quay is running.
# For example: https://myregistry.mycompany.com
# Example: http://quay.redhat2.cccr.local
            registry: http://quay.redhat2.cccr.local/keys/
  verifier_proxies:
  - enabled: true
    listen_addr: :6060
# If Clair is to be served via TLS, uncomment these lines. See the "Running Clair under TLS"
# section below for more information.
# key_file: /clair/config/clair.key
# crt_file: /clair/config/clair.crt
    verifier:
# CLAIR_ENDPOINT is the endpoint at which this Clair will be accessible. Note that the port
# specified here must match the listen_addr port a few lines above this.
# Example: https://clair.redhat2.cccr.local:6060
      audience: http://clair.redhat2.cccr.local:6060
      upstream: http://localhost:6062
      key_server:
        type: keyregistry
        options:
# QUAY_ENDPOINT defines the endpoint at which Quay is running.
# Example: https://myregistry.mycompany.com
# Example: http://quay.redhat2.cccr.local
          registry: http://quay.redhat2.cccr.local/keys/
```

- security_scanner.pem 파일 업로드

    인증서 업로드 위치 : /var/lib/clair-config/security_scanner.pem

- Clair 관련 파일 설정

```bash
# config.yaml
chcon -Rv -u system_u *.yaml

# security_scanner.pem
chcon -Rv -u system_u *.pem (security_scanner.pem)

# Context Change
chcon -Rv -t container_file_t *
```

- Clair Container 실행

```bash
docker run -d --restart=always -p 6060:6060 -p 6061:6061 \
-v /var/lib/clair-config:/clair/config \
quay.io/redhat/clair-jwt:v3.2.1
```

## Quay 웹 콘솔에서 Docker Build Test

- Docker Image Pull, Tag, Push to the [quay.redhat2.cccr.local](http://quay.redhat2.cccr.local)

```bash
# 외부 Internet에서 ubi7 Image Pull
docker pull ubi7

# tagging 작업
docker tag ubi7 quay.redhat2.cccr.local/admin/ubi7:v0.1

# Image Push
# Domain Name : quay.redhat2.cccr.local
# Namespace : admin
# Repository : ubi7
# Version : v0.1
docker push quay.redhat2.cccr.local/admin/ubi7:v0.1

The push refers to a repository [quay.redhat2.cccr.local/admin/ubi7]
48cf05bc9e5b: Pushed 
7eb0eb5e80b4: Pushing [====================================>              ] 150.9 MB/205.3 MB7eb0eb5e80b4: Pushed 
v0.1: digest: sha256:730177ee1a2da5169d2f3b147fa0c745f1d90b0094aafdab53678b834aed6d43 size: 737
```

- Robot 계정 생성

![Untitled 21](https://user-images.githubusercontent.com/67780144/96406821-26d76780-121b-11eb-8dc4-416d1620f860.png)

- Robot에게 ubi7 Repository에 대한 Read Permission 부여

![Untitled 22](https://user-images.githubusercontent.com/67780144/96406823-276ffe00-121b-11eb-86c6-1ee95d9b56bd.png)

- Build Trigger 클릭

![Untitled 23](https://user-images.githubusercontent.com/67780144/96406826-28089480-121b-11eb-832e-950ffd05cc66.png)

- Select file → Dockerfile 선택

    Local Host에서 작업해주어야 한다. 웹 콘솔에서 Dockerfile을 선택해야 하기 때문이다.

```bash
### Dockerfile

# Pulling할 Image
FROM quay.demo.com/admin/ubi7:v0.1

#RUN - Runs a command in the container
RUN echo "Hello world" > /tmp/hello_world.txt

#CMD - Identifies the command that should be used by default when running the image as a container.
CMD ["cat", "/tmp/hello_world.txt"
```

- Dockerfile Build

![Untitled 24](https://user-images.githubusercontent.com/67780144/96406828-28a12b00-121b-11eb-8e46-b0f858fe79a1.png)

- DockerBuild Successfully Done

![Untitled 25](https://user-images.githubusercontent.com/67780144/96406829-28a12b00-121b-11eb-935c-14c6b5fd0215.png)

- Repository Tage
    - 톱니바퀴 Click 후 Tag 이름 v0.2로 수정

![Untitled 26](https://user-images.githubusercontent.com/67780144/96406830-2939c180-121b-11eb-9e31-2a041bd8fedc.png)

- Tag History 확인

![Untitled 27](https://user-images.githubusercontent.com/67780144/96406831-29d25800-121b-11eb-8428-e6c6bc13d6bb.png)

- 수정되었는지 Check

```bash
[root@quay ~]# docker run -it quay.redhat2.cccr.local/admin/ubi7:v0.2
Hello world

[root@quay ~]# docker ps -a
CONTAINER ID        IMAGE                                             COMMAND                  CREATED             STATUS                     PORTS                                                                  NAMES
268c03123b2a        quay.redhat2.cccr.local/admin/ubi7:v0.2           "cat /tmp/hello_wo..."   2 minutes ago       Exited (0) 2 minutes ago                                                                          infallible_wozniak
```

## Openshift 내부 Registry에서 Quay Image Build

<< 여기서부터 써야함 >>