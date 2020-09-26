# Disconnected 설치 전 준비사항

# 1. 목적

- 본 프로젝트는 Disconnected 환경에서 인프라를 구축할 때 인터넷이 안되는 환경에서도 평소와 다름 없이 패키지와 Image 파일들을 사용하기 위해 아래의 작업을 진행해주었습니다.

# 2. 작업 내용

![Untitled](https://user-images.githubusercontent.com/67780144/94328282-249f3600-ffec-11ea-86e2-4b208fba61fb.png)

# 3. OCP 설치를 위해 필요한 것들

### URL 주소

- **RHEL 7.8 ISO 파일 다운로드** (Bastion 노드 설치할 때 사용)

[https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.8/x86_64/product-software](https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.8/x86_64/product-software)

- **Redhat CoreOS 파일 다운로드** (bootstrap, master, worker에 사용)

    [https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.4/4.4.17/](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.4/4.4.17/)

- **openshift-client**

    [https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.4.17/](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.4.17/)

### USB에 담아가야 할 파일들

1. **oc cli 명령어**

    openshift-client-linux-4.4.17.tar.gz

2. **Disconnected 환경 구축에 필요한 파일**
    - openshift-install-linux-4.4.17.tar.gz

        → Image Registry 생성 후 Binary 파일을 만들 수 있음

    - 패키지 다운을 위한 repository 압축 파일 (repos.tar.gz)

    - OCP 4 배포시 필요한 이미지 압축 파일 (data.tar.gz)

    - 레지스트리 컨테이너 이미지 (registry.tar)

    - rhcos-4.4.17-x86_64-installer.iso (RHCOS VM 생성 시 필요)

    - rhcos-4.4.17-x86_64-metal.raw.gz (RHCOS 부팅화면에서 기입)

    ++ operator 파일 ( 아직 진행 X )

# 4.(인터넷이 되는 환경에서)YUM Repository 준비

## 순서

- /etc/hosts에 domain 추가
- subscription 등록
- 사용할 Repository 등록
- yum-utils, createrepo 설치
- 사용할 Repo를 외부로부터 동기화 작업
- tar.gz 압축

## 과정

- /etc/hosts에 domain name 추가

```jsx
<host-ip> bastion.redhat2.cccr.local
```

- subscription 등록 후 Repository 등록

```bash
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

subscription-manager register

subscription-manager refresh

subscription-manager list --available --matches '*OpenShift*'

subscription-manager attach --pool=<pool_id>
subscription-manager repos --disable="*"

subscription-manager repos \
--enable="rhel-7-server-rpms" \
--enable="rhel-7-server-extras-rpms" \
--enable="rhel-7-server-ose-4.4-rpms"
```

- Install packages

    yum 패키지매니저 관련 유틸리티

    Repo 생성 패키지

```bash
yum -y install yum-utils createrepo
```

- repo shell script

    Enable한 Repo들을 동기화한 후 createrepo 명령어를 통해 만든다.

```bash
# vim  repo.sh
#! /bin/bash
mkdir /var/repos
for repo in \
rhel-7-server-rpms \
rhel-7-server-extras-rpms \
rhel-7-server-ose-4.7-rpms
do
 reposync -n --gpgcheck -l --repoid=${repo} --download_path=/var/repos
 createrepo -v /var/repos/${repo} -o /var/repos/${repo}
done

# sh repo.sh
```

- tar 압축

```bash
# tar 압축
tar cvfz repos.tar.gz /var/repos/*
```

# 5. (인터넷이 되는 환경에서)Image Registry

## 순서

- oc cli 설치
- podman, httpd-tools 설치
- TLS 인증서 작업 → 레지스트리 등에 사용
- htpasswd 명령어로 id,password 생성 → 레지스트리에 사용
- Image-registry 컨테이너 생성
- ocp 4.4 설치에 필요한 파일을 외부에서 Image registry로 복사
- openshift-install 바이너리 파일 생성
- 필요한 파일들 tar 압축

## 과정

- oc cli install

[https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.4.17/](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.4.17/)

```bash
# yum -y install wget

# wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.4.17/openshift-client-linux.tar.gz

# tar -xvf openshift-client-linux.tar.gz
README.md
oc
kubectl

# cp ./oc /usr/local/bin/
# cp ./kubectl /usr/local/bin/
```

- version check

```bash
# oc version
Client Version: 4.4.17

# kubectl version
~~ 1.17.0-4 ~~
```

- 필요 Package 설치

    ( httpd-tools : htpasswd 프로그램 제공, podman : container package )

```bash
yum -y install podman httpd-tools
```

- 작업 directory 생성

```bash
mkdir -p /opt/registry/{auth,data,certs}
```

- Certificate 작업

```bash
cd /opt/registry/certs

openssl req -newkey rsa:4096 -nodes -sha256 \
-keyout domain.key -x509 -days 3650 -out domain.crt

Generating a 4096 bit RSA private key
............................................................................................................................................................................................................++
...................................................................................................++
writing new private key to 'domain.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:KR 
State or Province Name (full name) []:seoul 
Locality Name (eg, city) [Default City]:seoul 
Organization Name (eg, company) [Default Company Ltd]:cccr
Organizational Unit Name (eg, section) []:student
Common Name (eg, your name or your server's hostname) []:bastion.redhat.cccr.local
Email Address []:hyukjin1994@gmail.com

cp /opt/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/

update-ca-trust extract
```

- jq 설치

```bash
cd /opt

curl -L https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64 -o /usr/local/bin/jq

chmod a+x /usr/local/bin/jq

jq -V
jq-1.6
```

- base64로 인코딩 된 user,password token 생성

```bash
htpasswd -bBc /opt/registry/auth/htpasswd devops dkagh1.

echo -n 'devops:dkagh1.'| base64 -w0
```

- Pull Secret 생성

    [https://cloud.redhat.com/openshift/install/metal](https://cloud.redhat.com/openshift/install/metal)

    ( 24시간 내에 설치를 완료해야 Secret을 정상적으로 적용시킬 수 있다. )

```bash
# cd /opt/registry

# vim pull-secret
{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfMWJhMGEz
..
..
0eW9RMTRObnlwT0xzajM1c1pXRGNac3lBcnJjSFQwQUVyRGc=","email":"xogns556@naver.com"}}}

# cat pull-secret | jq . > ./pull-secret.json

# vim pull-secret.json
{
  "auths": {
    "cloud.openshift.com": {
      "auth": "b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfMWJhMGEzM2Q1MjQzNDYyYmJhOTlhYWVmMGE5MWQ4ZTI6UTVLVU1LM0IyS0ozM0ZHSlRTMVRMQ1hHSjlBRkxFV0FZV0dMN0M5MDRVMElJRFU5RjlDSlc0UVhYRkcxUUo2Rg==",
      "email": "xogns556@naver.com"
    },
    "quay.io": {
      "auth": "b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfMWJhMGEzM2Q1MjQzNDYyYmJhOTlhYWVmMGE5MWQ4ZTI6UTVLVU1LM0IyS0ozM0ZHSlRTMVRMQ1hHSjlBRkxFV0FZV0dMN0M5MDRVMElJRFU5RjlDSlc0UVhYRkcxUUo2Rg==",
      "email": "xogns556@naver.com"
    },
    "registry.connect.redhat.com": {
      "auth": "fHVoYy0xZ3BTVEtvZ1pQY2JoV3FhZjF3aVl5TXRyeFc6ZXlKaGJHY2lPaUpTVXpVeE1pSjkuZXlKemRXSWlPaUpsTlRreU1UVmpZVEUxTnprME16azNPVEUyTVdJek56WTBNRFZqWXpNMVlTSjkucWZrSGZpSWk0ZDFnbmtJclFIQkZrVFZtZDdMYzVkZkFOTmNRbVV0MkJRcDVrLVR4MmhoUWk1bzFKeFg0cFZDYW95MWh0a0FGR3BnelBHYmxPYUc5ZTdVNWZVbVVSSW5MNE4wVTEzcHJTRGRRbUlwVDlia2FfNG0xZUs4YjJ2T29CcjBZbklWNHZHR2gxQTc5UThIOWJTMW5CMTdSUERTbWJfcml1RUxoazV5Q3l3ZzBvVTcyeXRuaU56dl9YaEtvQXB0YVBpNDFmOHVQbS1DVFJfMHFsd0VzOUtjVHMzVjdRbXByX3hKSko4dVMwZDBOalg1QTJtZmQ1c3N6azYzVzQ5Qm0yYmRiNVV5MnpGRWlCQnNEd1Z6eVBNbmNGYXBfV0VEaGs3R2JCemxBY3hrR1JuNkNuMVljVFFqZHEtRU8wcjNZOWJuSjg1Vk5Fak1kalhucy0xLVFkNVh4YUFyQ0FHa0NwWHp3aWtrYW90cXVTbTZwVUdMM0JiMGVQV0x2Wk1xMnVZaWs3cnY2UTdtYjNoaGs3TEJYQ2czSmRNNU9yRUMzVlBjU1dUOFdvZTZwOGQtYnhPUW9SR2toMjcwdlY0UEVjLTBMUGlDWnhyUEc2ZkdYbUxlZEdJbFpfdUlET1RYT1hWSkhmZWtpVkdROWZuQ3hSXzc1bDU4b0tnWHVQZWlLTzNPLW5lXzJKR3BrbFZBVmVqTDZWSHlxWG1XU0NFU0R1b05lQjNZWDZNUWNzX2w5WHQ0UmpUcjFWaWRud2xqaGNMUUFTLWxHeW44dUxkQ2pFNHVkS0lORzQzN2IyN0tiUHNzYkJIRzk2UVZIcVppaDktUWFlR2hvcTQ3QjJoQmxad2x0eW9RMTRObnlwT0xzajM1c1pXRGNac3lBcnJjSFQwQUVyRGc=",
      "email": "xogns556@naver.com"
    },
"bastion.redhat2.cccr.local:5000": { 
      "auth": "<<paste-here>>", 
      "email": "hyukjun1994@gmail.com"
  },
    "registry.redhat.io": {
      "auth": "fHVoYy0xZ3BTVEtvZ1pQY2JoV3FhZjF3aVl5TXRyeFc6ZXlKaGJHY2lPaUpTVXpVeE1pSjkuZXlKemRXSWlPaUpsTlRreU1UVmpZVEUxTnprME16azNPVEUyTVdJek56WTBNRFZqWXpNMVlTSjkucWZrSGZpSWk0ZDFnbmtJclFIQkZrVFZtZDdMYzVkZkFOTmNRbVV0MkJRcDVrLVR4MmhoUWk1bzFKeFg0cFZDYW95MWh0a0FGR3BnelBHYmxPYUc5ZTdVNWZVbVVSSW5MNE4wVTEzcHJTRGRRbUlwVDlia2FfNG0xZUs4YjJ2T29CcjBZbklWNHZHR2gxQTc5UThIOWJTMW5CMTdSUERTbWJfcml1RUxoazV5Q3l3ZzBvVTcyeXRuaU56dl9YaEtvQXB0YVBpNDFmOHVQbS1DVFJfMHFsd0VzOUtjVHMzVjdRbXByX3hKSko4dVMwZDBOalg1QTJtZmQ1c3N6azYzVzQ5Qm0yYmRiNVV5MnpGRWlCQnNEd1Z6eVBNbmNGYXBfV0VEaGs3R2JCemxBY3hrR1JuNkNuMVljVFFqZHEtRU8wcjNZOWJuSjg1Vk5Fak1kalhucy0xLVFkNVh4YUFyQ0FHa0NwWHp3aWtrYW90cXVTbTZwVUdMM0JiMGVQV0x2Wk1xMnVZaWs3cnY2UTdtYjNoaGs3TEJYQ2czSmRNNU9yRUMzVlBjU1dUOFdvZTZwOGQtYnhPUW9SR2toMjcwdlY0UEVjLTBMUGlDWnhyUEc2ZkdYbUxlZEdJbFpfdUlET1RYT1hWSkhmZWtpVkdROWZuQ3hSXzc1bDU4b0tnWHVQZWlLTzNPLW5lXzJKR3BrbFZBVmVqTDZWSHlxWG1XU0NFU0R1b05lQjNZWDZNUWNzX2w5WHQ0UmpUcjFWaWRud2xqaGNMUUFTLWxHeW44dUxkQ2pFNHVkS0lORzQzN2IyN0tiUHNzYkJIRzk2UVZIcVppaDktUWFlR2hvcTQ3QjJoQmxad2x0eW9RMTRObnlwT0xzajM1c1pXRGNac3lBcnJjSFQwQUVyRGc=",
      "email": "xogns556@naver.com"
    }
  }
}
-> 빨간 부분 추가

→ image registry Domain Name, Port 기입

→ 인증 : user,password 토큰 값

→ email : Certificate에 기입한 email 기입
```

- Open firewall port

```bash
firewall-cmd --add-port=5000/tcp

firewall-cmd --add-port=5000/tcp --permanent

firewall-cmd --reload
```

- registry 컨테이너 생성

```bash
podman run --name mirror-registry -p 5000:5000 \
-v /opt/registry/data:/var/lib/registry:z \
-v /opt/registry/auth:/auth:z \
-v /opt/registry/certs:/certs:z \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
-e REGISTRY_COMPATIBILITY_SCHEMA1_ENABLED=true \
-d docker.io/library/registry:2

curl -u devops:dkagh1. -k https://bastion.cccr.local:5000/v2/_catalog
{"repositories":[]}
```

- registry 복사 (Mirroring)

    맨 아래 부분 저장해놓기

```bash
# vi /opt/registry/mirror.sh

OCP_RELEASE=4.4.17-x86_64
LOCAL_REGISTRY='bastion.redhat2.cccr.local:5000'
LOCAL_REPOSITORY=ocp4/openshift4
PRODUCT_REPO='openshift-release-dev'
RELEASE_NAME="ocp-release"
LOCAL_SECRET_JSON=/opt/registry/pull-secret.json

oc adm -a ${LOCAL_SECRET_JSON} release mirror \
--from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE} \
--to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
--to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}

# sh -x /opt/registry/mirror.sh
..
..
info: Mirroring 109 images to bastion.cccr.local:5000/ocp4/openshift4 ...
bastion.cccr.local:5000/
  ocp4/openshift4
    blobs:
      quay.io/openshift-release-dev/ocp-v4.0-art-dev sha256:86496f2391f92e35e58ac09a6fd7b706a6d933d8d207edc68c36416dd5a51f8b 629B
..
..
..

**imageContentSources:
- mirrors:
  - bastion.redhat2.cccr.local:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - bastion.redhat2.cccr.local:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev**
```

- openshift-install 바이너리 파일 생성

```bash
oc adm -a ${LOCAL_SECRET_JSON} release extract --command=openshift-install "${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}"
```

- 이미지 압축

```bash
tar -cvfz data.tar.gz /opt/registry/data
```

- Image Registry 구성을 위한 Image 준비

```bash
podman images

podman save -o registry.tar docker.io/library/registry:2
```

# 6. (폐쇠망) Disconnected 환경에서 작업

- 폐쇠망에서는 인터넷을 통해 http 등의 패키지 조차 설치할 수 없으므로 가져온 repo 디렉터리를 통해 패키지 설치를 할 수 있도록 설정해주어야 한다.

![Untitled 1](https://user-images.githubusercontent.com/67780144/94328284-25d06300-ffec-11ea-9014-0b9a3adc5837.png)


## 1. Yum Repository

- (폐쇠망) repos.tar.gz 압축 푼 후 디렉터리 baseurl로 지정

```bash
cd /etc/yum.repos.d/

# vim local.repo
[base]
name=local repo
baseurl=file:///var/www/html/repo
enabled=1
gpgcheck=0
```

## 2. Image Registry

- data.tar.gz 파일 압축 풀기

- auth, certs 디렉터리도 생성한 후 위와 동일하게 나머지 작업

    (인증, 인증서 생성)

- registry.tar → Bastion으로 옮긴 후

```bash
podman load -i registry.tar

podman images
REPOSITORY           TAG               IMAGE ID            CREATED        SIZE
...                  ...               2d4f....

=> Image Hash 값 알아내기
```

- Registry 구축하기

```bash
podman run --name mirror-registry -p 5000:5000 \
-v /opt/registry/data:/var/lib/registry:z \
-v /opt/registry/auth:/auth:z \
-v /opt/registry/certs:/certs:z \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
-e REGISTRY_COMPATIBILITY_SCHEMA1_ENABLED=true \
-d 2d4
```

- Test

```bash
curl -u devops:dkagh1. -k https://bastion.cccr.local:5000/v2/_catalog
{"repositories":[]}
```
