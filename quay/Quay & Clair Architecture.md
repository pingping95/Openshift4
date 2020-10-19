# Quay & Clair Architecture

# 배포하기 전에 고려해야 할 것들

- Quay 아키텍쳐 패턴
    - 어떤 인프라에 Quay가 실행되는가?
    - 온프레미스? 퍼블릭 클라우드?
    - 어떠한 데이터베이스 서비스를 사용하는가?
    - 어떠한 스토리지 Backend를 사용하는가?
    - 별개의 레지스트리를 사용하는가? 공유 레지스트리를 사용하는가?
    - 폐쇄망에서 사용하는가?
    - Clair나 Builder가 사용되는가?

- Quay 배포 패턴
    - 자립형 Host vs 오픈쉬프트 / k8s
    - 타켓 Destination : 얼마나 많고 어디에 있는지?
    - Geo-Replication이 필요한지?
    - 컨텐츠 Ingress / Repo 미러링 Tree ?
    - Sizing
    - Subscription이 필요한가?
    - 고가용성이 필요한가?

# Quay Architecture

![Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled.png](Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled.png)

- On-Premise든 Public Cloud든 어떤 Infrastructure와 호환된다.
- 하지만 Openshift 에서 사용하는 것이 가장 권장된다.

### ++ Public Cloud에서의 Architecture

- AWS

![Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%201.png](Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%201.png)

- 2개의 가용 영역 (Available Zone)으로 고가용성을 구현할 수 있으며 Front 단에 AWS ELB를 두어 로드밸런싱을 한다.
- Backend 단에는 RDB, ElasticCache Redis, S3를 두어 백단을 구성한다.

# Prerequisite : Backend

- Database
    - 만약 이미지 보안 스캐닝을 위한 Clair을 사용하지 않을 경우에는 Database로 MySQL이나 MariaDB를 사용하면 되지만 QUAY - CLAIR Stack을 사용할 경우에는 PostgreSQL을 CLAIR에 사용해주어야 한다.
    - Public Cloud에서는 PostgreSQL의 사용이 권장된다.
- Storage
    - DB는 Binary Block형태의 데이터들을 저장하기 때문에 오브젝트 형태의 데이터들을 저장할 수 있는 스토리지가 필요하다.
    - NFS나 Local Storage는 지원하지 않음!
    - On-Premise에서는 Ceph Rados RGW, Openstack Swift, RHCOS 4,3 via NooBaa
    - Public Cloud에선 S3, Google Cloud Storage, Azure Blob Stogage등을 지원한다.

# 개별의 Registry (Dev & 공유)

- Dev와 Prod 사이에 명확한 경계
    - Organizations, Repositories, RBAC 권한을 통해 해결할 수 있다.
- Internal과 External 컨텐츠 사이의 명확한 경계
    - 위와 동일함

    → 결론적으로 Dev와 Prod 간의 영역을 명확하게 구분할 수 있다.

# Disconnected / Air-Gapped 환경에서의 Quay

![Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%202.png](Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%202.png)

![Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%203.png](Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%203.png)

- 첫 번째 Architecture는 현재 Quay Release
    - Quay가 외부 Source 레지스트리 (operatorhub.io나 RedHat Container Catalog, ..) 으로부터 Image들을 받아와 Mirroring하고 Clair을 통해 CVE Metadata를 Fetch한다.
    - Quay와 Openshift Cluster는 중간에 방화벽이 구성되어 OCP Node들은 외부 인터넷으로부터 Image를 Pull할 수 없고 오직 Quay로부터 이미지를 받아올 수 있다.
- 두 번째 Architecture는 향후 Quay Release이다.
    - QUAY와 Clair를 내부 폐쇄망에서 구현할 수 있도록 한다.
    - Export - Transfer - Import 형태이며 Blobs와 CVE data 둘 다 다룬다.

# Clair Overview

![Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%204.png](Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%204.png)

- Application Container들에 대한 취약성을 검사하는 정적 분석 Tool Open Source이다.
- Quay를 위해 CoreOS에 의해 개발되었다.
- 다양한 다른 프로젝트, 제 3자 의해 널리 사용된다.

## Quay Build Trigger

- Quay Build Trigger를 구현할 수 있다.
    - 자동으로 Build Trigger가 Dockerfile들을 Biuild한다.

        ⇒ 자동으로 Repository에 있는 Image들이 Push 액션이 발생하면 Build하며 Github, Bitbucket 등에 통합시킨다.

    - 동작 방식
        - Admin Panel로 첫 번째로 작동되어야 한다.
        - docker.socket에 마운트되어야 한다.
        - docker build 명령어와 상응된다.
        - Robot accounts to lock down automated access and audit each deployment (해석 불가)

# Repo Mirroring

![Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%205.png](Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%205.png)

- Whitelisted 컨텐츠를 Quay로 얻어오기
- 단일 Trust Source
    - 외부의 Source로부터 컨텐츠들을 명확하게 Whitelisted 한다.
    - HA Setup Geo-Replication Web_hooks

# Repo Mirroring - Registry Mirroring Tree

![Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%206.png](Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%206.png)

# Quay Recommendations

![Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%207.png](Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%207.png)

![Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%208.png](Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%208.png)

# Red Hat Quay 고가용성

![Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%209.png](Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%209.png)

- HAProxy 등을 통한 고객사의 HA 혹은 Load Balancing
- Stateless Quay 컴포넌트 고가용성 k8s나 Openshift Auto-Healing) 권장 사항 : 적어도 3개의 Quay, Clair, Mirroring 파드들
- Stateful한 Backend 컴포넌트들에 대한 HA 관리
- systemd (Linux에서의 최상위 Process)나 Kubernetes / Openshift를 통한 런타임 레벨에서의 고가용성
- 가상화 혹은 클라우드 플랫픔을 통한 Infrastructure 수준의 고가용성

# Storage Backend를 위한 HA - RHCOS 4

![Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%2010.png](Quay%20&%20Clair%20Architecture%20d91c1c0b50054cf08a4bc709abab5e93/Untitled%2010.png)

- 용어 정리

👉 CVE(Common Vulnerabilities and Exposures) : 공개적으로 알려진 정보 보안 결함 목록

👉 RDB : AWS 관계형 데이터베이스 서비스

👉 ElasticCache Redis : Redis와 호환되는 In-Memory 데이터 스토어 서비스

👉 S3 (Simple Storage Service) : 오브젝트 스토리지 서비스

👉 Geo-Replication : Single, Globally 분산되어 있는 Red Hat Quay Setup spread across multiple Datacenters / regions