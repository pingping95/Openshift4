# Openshift 4.4 Architecture


# 프로젝트 주요 목표

## 1. PaaS 클라우드 구축

보통 고객사들은 자사의 보안을 위해 외부와 Network 통신이 안되는 폐쇠망 환경에서 Cloud Platform 구축합니다.

이 때문에 PaaS 시장의 일반적인 환경인 Disconnected 환경에서 기본 Infra 환경을 설계하고 구축하기 위한 사전 준비 사항 및 고려 사항을 파악하여 Internet이 단절된 환경에서의 PaaS 클라우드 아키텍쳐 수립 및 환경 구축을 하고자 합니다.

## 2. DevSecOps Pipeline 구성

개발, 배포 및 보안을 위한 DevSecOps Pipeline 구성의 필요성과 Jenkins, Gitlab, API, Quay 등을 연계하여 환경 및 데이터 보안, CI/CD 프로세스 보안을 고려한 Pipeline을 구성하고 PaaS 클라우드 환경에서 빌드 및 배포의 자동화를 구성하고자 합니다. 

# 주요 내용

- Disconnected 환경에서의 기초 인프라 Architecture 설계 및 구축
    - 기반 서비스 (DNS, Haproxy, Chrony, Yum Repository, Image-Registry) 설계와 구축
    - 애플리케이션 서비스 (Gitlab, Image Registry) 설계와 구축
- 표준 컨테이너 및 오케스트레이션 기술 이해
- 프라이빗 PaaS 클라우드 (Openshift 4) 아키텍쳐 설계 및 구축
    - Master, Infra, Router, App Node 설계와 구축
- Git, Jenkins, Quay 등을 활용한 DevSecOps Pipeline 설계와 구축
    - 사용자 ID 권한 부여 및 Access 제어
    - 컨테이너 분리 및 네트워크 분리
    - 컨테이너 이미지 스캔

# 인프라 구성 환경

## 프로젝트 환경

- 노트북 4대, Hub 1대,  LAN Cable 4개
    - 노트북 스펙
        - Ubuntu 18.04
        - 5 Core
        - RAM 16 GB
        - SSD 512 GB
        - Hypervisior : KVM
- 네트워크 통신 방법
    - Host < — > 다른 Host의 VM : Bridge 통신
    - Host < — > 본인 Guest VM : Host-Only 통신
- 원격 방안
    - 교육장 내에 자체 Center 구축
    - Team Viewer 접속

![Untitled](https://user-images.githubusercontent.com/67780144/94980867-8c173180-0568-11eb-9cad-f8099fe1c56e.png)


# Architecture

## 1. 각 노드별 Resource, IP Settings, 역할

- Cluster Name : redhat2
- Base Domain : [cccr.local](http://cccr.local)

![Untitled 1](https://user-images.githubusercontent.com/67780144/94980868-8d485e80-0568-11eb-8861-7382d276da11.png)


- Bastion Node : Openshift 4 Container Platform 구축의 기반 역할
    - DNS, HAProxy, Image Registry, Chrony, Yum Repository 서버 구축

- Master Node : Openshift 4 Control Plane Node, 고가용성을 위한 3중화

- Router Node : Application이 배포될 Service Node로 Routing Node

- Infra Node : Logging, Monitoring 를 위한 Node

- Service Node : 실질적인 Application이 배포되는 Node

## 2. 논리적 Architecture

![Untitled 2](https://user-images.githubusercontent.com/67780144/94980870-8de0f500-0568-11eb-8f6f-9279f8914736.png)


## 3. 물리적 Architecture

- Laptop 4대에 VM의 리소스 할당량을 고려하여 배치하였습니다.
- Com #3의 경우 Bootstrap은 Master Node 설치 후 삭제해도 되므로 삭제한 후 Infra #2, Router Node를 올렸습니다.

![Untitled 3](https://user-images.githubusercontent.com/67780144/94980871-8e798b80-0568-11eb-9a0e-3328f0ac4960.png)

