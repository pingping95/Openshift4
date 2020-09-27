# Disconnected 환경에서 OLM 사용

# 목적

- 기본 Source를 비활성화하고 OLM (Openshift Lifecycle Management)이 Local에서 Operator를 대신 설치 및 관리할 수 있도록 Local Mirror을 만드는 것

# Operator Catalog Image 이해

- OLM은 항상 최신 버전의 Operator Catalog에서 Operator을 설치한다. OCP 4.3부터 Redhat에서 제공하는 Operator는 quay.io에서 Quay App Registry 카탈로그를 통해 배포된다.

```jsx
redhat-operators : Red Hat에서 패키지 및 배송 한 Red Hat 제품에 대한 공개 카탈로그. Red Hat에서 지원합니다.

certified-operators : 주요 ISV (독립 소프트웨어 공급 업체)의 제품에 대한 공개 카탈로그. Red Hat은 ISV와 협력하여 패키징 및 배송합니다. ISV에서 지원합니다.

community-operators : 운영자 프레임 워크 / 커뮤니티 운영자 GitHub 저장소 에서 관련 담당자가 유지 관리하는 소프트웨어 용 공개 카탈로그입니다 . 공식적인 지원이 없습니다.
```

- Catalog가 업데이트되면 최신 버젼의 Operator가 변경되고 이전 버젼이 제거되거나 변경될 수 있다.

    → 이러한 행위가 시간이 지남에 따라 재현 가능한 설치를 유지하는데 문제가 발생할 수 있다.

- OLM이 폐쇠망에서의 OCP 클러스터에서 실행되는 경우 quay.io에서 직접 카탈로그에 액세스할 수 없다.

- `oc adm catalog build`  명령을 사용하여 Cluster Operator는 Operator Catalog Image를 만들 수 있다.

# Operator Catalog Image 빌드

### 전제 조건

아래와 같은 전제 조건이 있다.

- oc 4.3.5 이상

```bash
oc version
```

- podman 1.4.4 이상
- 네트워크 액세스 가능
- [Quay.io](http://quay.io) 계정이 액세스할 수 있는 Private Namespace로 작업하는 경우 Quay 인증 토큰을 설정해야 한다.
- 개인 레지스트리로 작업하는 경우 : REG_CREDS 이후 단계에서 사용하기 위해 환경 변수를 레지스트리 자격 증명 파일 경로에 설정해야 한다. (pull-secret, 아래에서 나옴)

### 절차

- (폐쇠망일 경우) 타겟 Mirror Registry에 인증

```bash
podman login bastion.redhat2.cccr.local:5000
```

- redhat-operators Catalog 기반 카탈로그 이미지를 빌드

```bash
oc adm catalog build \
--fromappregistry-org redhat-operators \
--from=registry.redhat.io/openshift4/ose-operator-registry:v4.4 \
--to=bastion.redhat2.cccr.local:5000/olm/redhat-operators:v1 \
--insecure
-a /ocp/registry/pull-secret.json

## Options
[-a ${REG_CREDS}] \                    # Optional
[--insecure] \                         # Optional
[--auth-token "${AUTH_TOKEN}"]         # Optional
```

- 옵션들 설명
    - Organization (namespace) to pull from an App Registry instance.
    - Set --from to the ose-operator-registry base image using the tag that matches the target OpenShift Container Platform cluster major and minor version.
    - Set --filter-by-os to the operating system and architecture to use for the base image, which must match the target OpenShift Container Platform cluster. Valid values are linux/amd64, linux/ppc64le, and linux/s390x.
    - Name your catalog image and include a tag, for example, v1.
    - Optional: If required, specify the location of your registry credentials file.
    - Optional: If you do not want to configure trust for the target registry, add the --insecure flag.
    - Optional: If other application registry catalogs are used that are not public, specify a Quay authentication token.

# 폐쇠망을 위한 Operator Hub 구성

## 목적

- Cluster Operator가 Custom Operator Catalog Image를 사용하여 폐쇠망에서 Local 컨텐츠를 사용하도록 OLM 및 Operator Hub 구성을 목적으로 한다.

## 순서

- 기본 OperatorSources 비활성화

    OCP 설치 중에 기본적으로 구성되는 기본 OperatorSource가 비활성화 해준다.

```bash
oc patch OperatorHub cluster --type json \
-p '[{ "op": "add", "path": "/ spec / disableAllDefaultSources", "value": true}]'
```

- oc adm catalog mirror → 미러링에 필요한 매니페스트 생성을 위해 Custom Operator Catalog Image의 콘텐츠를 추출

```bash
oc adm catalog mirror \
bastion.redhat2.cccr.local:5000/olm/redhat-operators:v1 \
bastion.redhat2.cccr.local:5000 \
-a /ocp/registry/pull-secret.json

## Optionals
[--insecure] \
[--filter-by-os="<os>/<arch>"] \
[--manifests-only]
```

- Redhat-operators-manifests 디렉터리 내용 확인

    → 자신이 설정한 Mirror Registry 주소로 설정되었는지 내용 확인. 맞지 않을 경우 수정 해야함

```bash
# ls redhat-operators-manifests
mapping.txt
imageContentSourcePolicy.yaml

# cat imageContentSourcePolicy.yaml
# cat mapping.txt
```

- 생성된 manifests 파일 적용

```bash
oc apply -f ./redhat-operators-manifests
```

# CatalogSources 객체 생성

- catalogsources.yaml 작성

    ```bash
    # vi catalogsources.yaml

    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
      name: my-operator-catalog
      namespace: openshift-marketplace
    spec:
      sourceType: grpc
      image: registry.demo.ocp42.com:5000/olm/redhat-operators:v1
      displayName: My Operator Catalog
      publisher: grpc
    ```

- catalogsource 생성

    ```bash
    oc create -f catalogsource.yaml
    ```

- pod, catalogsource 확인

    ```bash
    oc get pods -n openshift-marketplace

    oc get catalogsource -n openshift-marketplace

    oc get packagemanifest -n openshift-marketplace
    ```

# OperatorHub 제공 Operator 설치 (Logging)

## 1. Elasticsearch Operators 설치

- 디렉터리 생성

    ```bash
    mkdir $HOME/cluster-logging
    ```

- Elasticsearch Operator 네임스페이스 작성

    ```bash
    # vi $HOME/cluster-logging/es_namespace.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: openshift-operators-redhat
      annotations:
        openshift.io/node-selector: ""
      labels:
        openshift.io/cluster-logging: "true"
        openshift.io/cluster-monitoring: "true"  

    ## es-namespace 객체 생성
    oc create -f $HOME/cluster-logging/es-namespace.yaml
    ```

- openshift-operators-redhat의 Operator-Group 생성

    ```bash
    # vi $HOME/cluster-logging/es-og.yaml
    apiVersion: operators.coreos.com/v1
    kind: OperatorGroup
    metadata:
      name: openshift-operators-redhat
      namespace: openshift-operators-redhat
    spec: {}

    ## es-og 객체 생성
    oc create -f $HOME/cluster-logging/es-og.yaml
    ```

- Elasticsearch-operaor의 사용 가능한 채널 검색

    ```bash
    oc get packagemanifest elasticsearch-operator -n openshift-marketplace -o jsonpath='{.status.defaultChannel}'
    ```

- 사용 가능한 채널에 맞춰 Elasticsearch Operator Subscription 생성

    ```bash
    # vi $HOME/cluster-logging/es-sub.yaml
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: elasticsearch-operator
      namespace: openshift-operators-redhat
    spec:
      channel: "4.3"
      installPlanApproval: "Automatic"
      name: elasticsearch-operator
      source: my-operator-catalog
      sourceNamespace: openshift-marketplace

    ## es-sub 객체 생성
    oc create -f $HOME/cluster-logging/es-sub.yaml
    ```

- 모든 Namespace에 CSV(Cluster Service Version) 생성 확인

    ```bash
    oc get csv -A

    NAMESPACE                                               NAME                                        DISPLAY                  VERSION              REPLACES   PHASE
    cicd-sample                                             elasticsearch-operator.4.3.5-202003020549   Elasticsearch Operator   4.3.5-202003020549              Succeeded
    default                                                 elasticsearch-operator.4.3.5-202003020549   Elasticsearch Operator   4.3.5-202003020549              Succeeded
    dev-sample                                              elasticsearch-operator.4.3.5-202003020549   Elasticsearch Operator   4.3.5-202003020549              Succeeded
    jenkins                                                 elasticsearch-operator.4.3.5-202003020549   Elasticsearch Operator   4.3.5-202003020549              Succeeded
    kube-node-lease                                         elasticsearch-operator.4.3.5-202003020549   Elasticsearch Operator   4.3.5-202003020549              Succeeded
    kube-public                                             elasticsearch-operator.4.3.5-202003020549   Elasticsearch Operator   4.3.5-202003020549              Succeeded
    kube-system                                             elasticsearch-operator.4.3.5-202003020549   Elasticsearch Operator   4.3.5-202003020549              Succeeded
    my-hpa      
    ….
    ```

- Prometheus에게 권한을 부여받기 위한 RBAC 작성, 생성

    ```bash
    # vi $HOME/cluster-logging/es-rbac.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: prometheus-k8s
      namespace: openshift-operators-redhat
    rules:
    - apiGroups:
      - ""
      resources:
      - services
      - endpoints
      - pods
      verbs:
      - get
      - list
      - watch
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: prometheus-k8s
      namespace: openshift-operators-redhat
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: prometheus-k8s
    subjects:
    - kind: ServiceAccount
      name: prometheus-k8s
      namespace: openshift-operators-redhat

    ## es-rbac 객체 생성
    oc create -f $HOME/cluster-logging/es-rbac.yaml
    ```

- Elasticsearch Operator 잘 되는지 확인

    ```bash
    oc get pods -n openshift-operators-redhat -o wide

    oc logs elasticsearch-operator-588498db47-mdkm8 -n openshift-operators-redhat
    ```

## Cluster Logging Operators 설치

- Cluster Logging Operator의 Namespace 작성, 생성

    ```bash
    # vi $HOME/cluster-logging/clo-namespace.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: openshift-logging
      annotations:
        openshift.io/node-selector: ""
      labels:
        openshift.io/cluster-logging: "true"
        openshift.io/cluster-monitoring: "true"

    ## clo-namespace 객체 생성
    oc create -f $HOME/cluster-logging/clo-namespace.yaml
    ```

- openshift-logging Operator Group 생성

    ```bash
    # vi $HOME/cluster-logging/clo-og.yaml
    apiVersion: operators.coreos.com/v1
    kind: OperatorGroup
    metadata:
      name: cluster-logging
      namespace: openshift-logging
    spec:
      targetNamespaces:
      - openshift-logging

    ## clo-og 객체 생성
    oc create -f $HOME/cluster-logging/clo-og.yaml
    ```

- cluster-logging Subscription 생성

    ```bash
    # vi $HOME/cluster-logging/clo-sub.yaml
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: cluster-logging
      namespace: openshift-logging
    spec:
      channel: "4.3"
      name: cluster-logging
      source: my-operator-catalog
      sourceNamespace: openshift-marketplace

    ## clo-sub 객체 생성
    oc create -f $HOME/cluster-logging/clo-sub.yaml
    ```

- CSV 잘 생성되었는지 확인

    ```bash
    oc get clusterserviceversions.operators.coreos.com -n openshift-logging
    ```

## Cluster Logging Instance 설치

- Cluster Logging Operator의 Instance를 작성하고 생성한다.

    → Cluster Logging Node의 갯수와 Memory, Storage 유의할 것

    → redundancyPolicy : 노드 구성에 따라 다르게 설정한다.

    참고 : [https://docs.openshift.com/container-platform/4.3/logging/cluster-logging-deploying-about.html#cluster-logging-deploy-storage-considerations_cluster-logging-deploying-about](https://docs.openshift.com/container-platform/4.3/logging/cluster-logging-deploying-about.html#cluster-logging-deploy-storage-considerations_cluster-logging-deploying-about)

    ```bash
    # vi clo-instalce.yaml
    apiVersion: "logging.openshift.io/v1"
    kind: "ClusterLogging"
    metadata:
      name: "instance"
      namespace: "openshift-logging"
    spec:
      managementState: "**Managed**"ss          ## **주의**
      logStore:
        type: elasticsearch
        elasticsearch:
          resources:
            **limits:
              memory: 6Gi                   ## 주의
            requests:                       ## 주의
              memory: 6Gi                   ## 주의**
          nodeCount: 1
          redundancyPolicy: ZeroRedundancy
          **storage:                          ## 주의
            storageClassName: standard      ## 주의
            size: 10Gi                      ## 주의**
      visualization:
        type: "kibana"
        kibana:
          replicas: 1
      curation:
        type: "curator"
        curator:
          schedule: "30 3 * * *"
      collection:
        logs:
          type: "fluentd"
          fluentd: {}

    ## clo-instance 객체 생성
    oc create -f clo-instance.yaml
    ```

- PV생성

    자동 생성된 pvc에 바인딩하기 위한 PV를 생성해주낟.

    ```bash
    oc get pvc             ## pv가 없어 pending 상태일 것임

    # vi elasticsearch-pv-1.yaml
    apiVersion: v1
    kind : PersistentVolume
    metadata:
      name : elasticsearch-cdm-pv1
    spec:
      capacity:
        storage: 10Gi
      accessModes:
      - ReadWriteOnce
      nfs:
        path: /pv/logging                           # 적절히 수정
        server: bastion.redhat2.cccr.local          # 수정
      PersistentVolumeReclaimPolicy: Retain
      storageClassName: standard
      claimRef:
        name: elasticsearch-elasticsearch-cdm-pxxc8cyk-1
        namespace: openshift-logging

    oc create -f elasticsearch-pv-1.yaml
    ```

- Pod 확인 (Operator, Elasticsearch, Kibana, Fluentd Pod)

    ```bash
    oc get pods -n openshift-operator -o wide
    ```

- 수집된 Log 확인

    ```bash
    ls -rtl /pv/logging/elasticsearch/logs
    ```

- Kibana에서 Log 확인 및 Dashboard 접속

    ```bash
    oc get route -n openshift-logging
    ```

