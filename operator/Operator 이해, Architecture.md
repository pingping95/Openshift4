# Operator 이해, Architecture

- 공식 문서 보고 정리하였습니다.

[Understanding Operators](https://docs.openshift.com/container-platform/4.4/operators/olm-what-operators-are.html)

- OLM 란?
    - cluster에서 실행되는 모든 operator 및 관련 서비스의 Lifecycle를 설치, 업데이트 및 관리를 담당
    - k8s Native Application (Operator)을 효과적이고 자동화하고 확장 가능한 방식으로 관리하도록 설계 된 Open Source Tool인 Operator Framework의 일부이다.

![Operator%20%E1%84%8B%E1%85%B5%E1%84%92%E1%85%A2,%20Architecture%202f6a97ee21ff4f2fba66f57f6e058400/Untitled.png](Operator%20%E1%84%8B%E1%85%B5%E1%84%92%E1%85%A2,%20Architecture%202f6a97ee21ff4f2fba66f57f6e058400/Untitled.png)

- OLM Workflow

    OCP 4.4에서 실행되며 Cluster Operator가 Cluster에서 실행중인 Operator에 대한 접근 권한을 설치, 업그레이드 및 부여하는데 유용하다.

# CSV (Cluster Service Versions)

- CSV는 Operator Metadata에서 생성된 YAML 매니페스트로서, OLM이 Cluster에서 Operator를 실행하는데 도움을 준다.
- CSV는 사용자 인터페이스를 로고, 설명 및 버전과 같은 정보로 채우는 데 사용되는 오퍼레이터 컨테이너 이미지에 수반되는 메타데이터다. 또한 RBAC Rules와 같이 Operator가 관리하거나 의존하는 Custom Resource(CR)를 실행하는 데 필요한 기술 정보의 원천이기도 하다.
- CSV 구성요소
    - Metadata : Name, description, version (semver compliant), links, labels, icon, etc.
    - Install strategy : Set of SA(Service Account) and required permissions, Set of Deployments...
    - CRD
        - Owned: Managed by this service
        - Required: Must exist in the cluster for this service to run
        - Resources: A list of resources that the Operator interacts with
        - Descriptors: Annotate CRD spec and status fields to provide semantic information

# OLM Operator 설치, 업그레이드 Workflow

- CSV
- CatalogSource
- Subscription

- CatalogSource 개요

![Operator%20%E1%84%8B%E1%85%B5%E1%84%92%E1%85%A2,%20Architecture%202f6a97ee21ff4f2fba66f57f6e058400/Untitled%201.png](Operator%20%E1%84%8B%E1%85%B5%E1%84%92%E1%85%A2,%20Architecture%202f6a97ee21ff4f2fba66f57f6e058400/Untitled%201.png)

---

- 출처

    호롤리님의 블로그 인용, 4.3 버전이지만 잘 정리되어 있어 가져왔다.

    [https://gruuuuu.github.io/ocp/ocp-control-plane/#](https://gruuuuu.github.io/ocp/ocp-control-plane/#)

## **Operators in OpenShift Container Platform**

**Openshift Container Platform**에서 Operator는 control plane에서 서비스를 관리하고 배포하며 패키징하는 역할을 담당합니다. `kubectl` 과 `oc` 커맨드와 같은 Kubernetes API와 CLI툴을 통합해주며, health check, 업데이트 관리, application이 지정된 상태를 유지하게 해주는 역할도 합니다.

CRI-O와 kubelet이 모든 노드에서 돌아가고 있기 때문에, 거의 모든 클러스터 기능은 operator로 사용할 수 있으며 control plane에 의해 관리됩니다.그런 맥락에서 ocp4.3의 컴포넌트중 가장 중요한 컴포넌트라고 할 수 있습니다.

### **PLATFORM OPERATORS IN OPENSHIFT CONTAINER PLATFORM**

ocp4.3에서는 모든 클러스터 기능이 `Platform Operator`로 구분됩니다.**Platform Operator**는 클러스터의 특정 영역의 기능을 관리합니다.

- 전 클러스터 application의 로깅
- control plane의 관리
- 머신 provisioning 시스템의 관리

각 operator는 간단한 API로 기능을 제공하며, 글로벌 구성파일을 수정하는 대신에 API를 수정함으로써 일반적인 작업을 자동화시켜 운영부담을 줄일 수 있습니다.

### **OPERATORS MANAGED BY OLM**

**Cluster Operator Lifecycle Management**(OLM)은 application에서 사용할 수 있는 operator를 관리하는 컴포넌트입니다. (ocp를 구성하는 operator는 관리대상이 아님)다시말해 kubernetes-native app을 operator로써 관리하는 일종의 프레임워크가 OLM입니다.

OLM은 `Red Hat Operator`와 `Certified Operator`로 나뉩니다.`Red Hat Operator`는 스케줄러나 etcd같은 클러스터 기능을 제공하고, `Certified Operator`는 community에서 빌드하고 관리하며, 전통적인 application에 API레이어를 제공해서 쿠버네티스기반으로 application을 관리할 수 있게 해주는 기능을 제공합니다.

### **ABOUT THE OPENSHIFT CONTAINER PLATFORM UPDATE SERVICE**

`ocp update service`는 Openshift Container Platform과 Red Hat Enterprixe Linux CoreOS(RHOCS)에 over-the-air(OTA) 업데이트를 제공해주는 서비스입니다.각 operator 컴포넌트끼리 연결된 그래프와 다이어그램을 제공해서 어떤 버전으로 업그레이드를 해야 안전하게 할 수 있는지, 업데이트 할 클러스터의 예상 상태를 보여줄 수 있습니다.

`Cluster Version Operator`(CVO)는 최신 컴포넌트의 버전과 정보를 바탕으로 update service가 적절한 업데이트를 찾았는지 체크하는 역할을 합니다. 유저가 업데이트를 진행하려고 할 때, CVO는 release된 이미지를 찾아서 클러스터를 업그레이드합니다.

update 서비스가 호환되는 버전의 업데이트만 제공하게 하려면, 검증 파이프라인이 존재해야합니다. 각 릴리즈 버전은 클라우드 플랫폼, 시스템 아키텍처, 기타 구성요소와의 호환성을 검증하고 파이프라인이 적합하다고 판단되면 update 서비스는 사용자에게 업데이트가 이용가능하다고 알려줍니다.

지속 업데이트 모드가 활성화되어있는 동안 두개의 컨트롤러가 동작합니다. 하나는 지속적으로 페이로드 매니페스트 파일을 업데이트하고 클러스터에 적용시키며 operator의 상태를 출력합니다. 두번째 컨트롤러는 update서비스를 통해 업데이트가 가능한 상태인지를 확인하는 역할을 합니다.

> 클러스터를 이전버전으로 되돌리는 것은 지원하지 않습니다. 반드시 새로운 버전으로만 업그레이드해야 합니다.

업그레이드 과정중에 `Machine Config Operator`(MCO)는 새로운 구성을 클러스터에 적용하는 역할을 합니다.

### **UNDERSTANDING THE MACHINE CONFIG OPERATOR**

ocp4.3은 운영체제와 클러스터 관리를 모두 통합해서 쓸 수 있습니다. 클러스터는 RHCOS를 포함한 자체 업데이트를 관리하므로 ocp는 노드 업그레이드를 단순화시키는 라이프사이클 관리 환경을 제공합니다.

노드 관리를 단순화하기 위해 3개의 데몬셋 및 컨트롤러를 사용합니다.

- `machine-config-controller` : control plane에서 머신 업그레이드를 조정하는 역할을 합니다. 모든 클러스터 노드를 모니터링하고 구성 업데이트를 오케스트레이션합니다.
- `machine-config-daemon` : 클러스터의 각 노드에서 실행되며 MachineConfig에 정의된 구성으로 머신을 업데이트 하는 역할을 합니다. 노드에 변경사항이 표시되면 pod을 drain(축출?)하고 업데이트를 적용한 후에 재부팅합니다. 이러한 변경사항은 시스템 구성 및 kubelet구성을 담은 ignition파일의 형태로 제공됩니다. 이 프로세스는 ocp및 RHCOS를 함께 관리하는데 핵심적인 요소로 사용됩니다.
- `machine-config-server` : 마스터노드가 클러스터에 조인할때 ignition파일을 마스터에 제공합니다.
