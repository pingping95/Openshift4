# Disconnected 설치 전 준비사항

- 보통 3 Master, 2 Infra를 많이 사용한다. 본 프로젝트에서는 Resource의 제약으로 인해 아래와 같은 Spec으로 프로젝트를 진행할 것이다.
- 

- **Bastion**
    - vCPU : 4
    - RAM : 8 GB
    - Storage : 100 GB
    - Data : 100 GB
- **BootStrap**
    - vCPU : 4
    - RAM : 16 GB
    - Storage : 120 GB
- **Master ( 3중화 )**
    - vCPU : 4
    - RAM : 8 GB
    - Storage : 100 GB
    - Container Runtime : 100 GB

- **Infra ( 2중화 )**
    - vCPU : 4
    - RAM : 8 GB
    - Storage : 100 GB
    - Container Runtime : 100 GB
- **Service ( 2중화 )**
    - vCPU : 2
    - RAM : 4 GB
    - Storage : 100GB
    - Container Runtime : 100GB
- **Router**
    - vCPU : 1
    - RAM : 2 GB
    - Storage : 100GB
    - Container Runtime : 100GB

# Check Lists

## 0. IP & FQDN Settings
- IP는 아래처럼 설정해줌

| Server Name | FQDN | IP Address |
|---|:---:|---:|
| Bootstrap | bootstrap.redhat2.cccr.local | 10.10.10.10 |
| Master #1 | master-1.redhat2.cccr.local | 10.10.10.11 |
| Master #2 | master-2.redhat2.cccr.local | 10.10.10.12 |
| Master #3 | master-3.redhat2.cccr.local | 10.10.10.13 |
| Infra #1 | infra-1.redhat2.cccr.local | 10.10.10.14 |
| Infra #2 | infra-1.redhat2.cccr.local | 10.10.10.15 |
| Router | infra-2.edhat2.cccr.local | 10.10.10.16 |
| bastion | bastion.edhat2.cccr.local | 10.10.10.17 |
| Service #1 | service-1.edhat2.cccr.local | 10.10.10.18 |
| Service #2 | service-2.edhat2.cccr.local | 10.10.10.19 |

## 1. Firewall

| TCP | 설명 |
|---|:---:|
| 2379-2380| etcd server, peer, and metrics ports |
| 6443 | Kubernetes API |
| 9000-9999 | Host level services, including node exporter(9100-9101) and the Cluster Version Operator(9099) |
| 10249-10259 | The default ports that Kubernetes reserves |
| 10256 | openshift-sdn |
| 8080 | Bastion Node`s Web file Server Port |

| UDP | 설명 |
|---|:---:|
| 9000-9999| Host level services, including the node exportet on ports 9100-9101 |
| 30000-32767 | Kubernetes NodePort |


## 2. DNS Records

- Kubernetes API

    ⇒ api.<cluster_name>.<base_domain>

    ⇒ api-int.<cluster_name>.<base_domain>

- Routes

    ⇒ .apps.<cluster_name>.<base_domain>

- etcd

    ⇒ etcd-.<cluster_name>.<base_domain>

    ⇒ _ etcd-server-ssl._tcp.<cluster_name>.<base_domain>

## 3. Disk & Network Interface 정보

- RHCOS mini-install하여 정보를 확인해보기

## 4. OCP 설치를 위한 파일 준비

- URL 주소
    - RHEL 7.7 ISO 파일 다운로드 (Bastion 노드 설치할 때 사용)

    [https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.8/x86_64/product-software](https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.8/x86_64/product-software)

    - CoreOS 파일 다운로드 (bootstrap, master, worker에 사용)

        [https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.4/4.4.17/](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.4/4.4.17/)

    - openshift-install, openshift-client (Bastion 노드에서 사용)

        [https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/)

- Static IP 배포할 때
    - rhcos-4.4.17-x86_64-installer.iso (설치할 때 필요)

    - rhcos-4.4.17-x86_64-metal.raw.gz (부팅화면에서 기입)

    - openshift-client-linux-4.4.17.tar.gz

    - openshift-install-linux-4.4.17.tar.gz

- Disconnected 환경에 가져갈 파일

    ○ 패키지 다운을 위한 repository 압축 파일 (repos.tar.gz)

    -

    ○ OCP 4 배포시 필요한 이미지 압축 파일 (data.tar.gz)

    -

    ○ Private registry 구성을 위한 이미지 (registry.tar)

# YUM Repository Repo 준비

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

    필요한 패키지 설치

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
 reposync -n  --gpgcheck -l --repoid=${repo} --download_path=/var/repos createrepo -v  /var/repos/${repo} -o /var/repos/${repo}   
done

# sh repo.sh
```

- tar 압축

```bash
# tar 압축
tar cvfz repos.tar.gz /var/repos/*
```

# Image Registry

- oc cli install

[https://cloud.redhat.com/openshift/install/metal](https://cloud.redhat.com/openshift/install/metal)

```bash
# yum -y install wget

# wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz

# tar -xvf openshift-client-linux.tar.gz
README.md
oc

# cp ./oc /usr/local/bin/
# cp ./kubectl /usr/local/bin/
```

- version check

```bash
[root@bastion ~]# oc version
Client Version: 4.5.7

[root@bastion var]# kubectl version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.2-0-g52c56ce", GitCommit:"e40bd2dd95be1106ead550990c549f932752b239", GitTreeState:"clean", BuildDate:"2020-09-04T13:51:53Z", GoVersion:"go1.13.4", Compiler:"gc", Platform:"linux/amd64"}
```

- 필요 Package 설치

    ( httpd-tools : htpasswd 프로그램 제공, podman : container package )

```bash
yum -y install podman httpd-tools
```

- 작업 directory 생성

```bash
mkdir  -p /opt/registry/{auth,data,certs}
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
Email Address []:xogns556@naver.com

cp  /opt/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/

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
echo -n 'devops:dkagh1.'| base64 -w0
dGVzdDoxMjM0
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
"bastion.redhat.cccr.local:5000": { 
      "auth": "dGVzdDoxMjM0", 
      "email": "xogns556@naver.com"
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

curl -u test:1234 -k https://bastion.cccr.local:5000/v2/_catalog
{"repositories":[]}
```

- registry 복사 (Mirroring)

    맨 아래 부분 저장해놓기

```bash
# vi /opt/registry/mirror.sh

OCP_RELEASE=4.4.10-x86_64
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

- 이미지 압축

```bash
tar -cvfz data.tar.gz /opt/registry/data
```

- Private Registry 구성을 위한 Image 준비

```bash
podman images

podman save -o registry.tar docker.io/library/registry:2
```
