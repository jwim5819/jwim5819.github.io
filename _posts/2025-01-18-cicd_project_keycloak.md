---
title: CI/CD 구축기 - 6. keycloak 설치
date: 2025-01-18 23:00:00 +09:00
categories: [Development, DevOps]
tags: [Kubernetes, Infra, DevOps, Keycloak, SSO]
---

## 🔒 서론

> IT인프라에서 인증(Authentication)과 권한(Authorization)은 뗄 수 없는 중요한 요소이다. 특히, 여러 어플리케이션이나 마이크로서비스를 운여할 땐 사용자 관리가 복잡하게 꼬이기 십상이다. 그럴 때 등장하는 게 keycloak이다.  
> keycloak은 오픈소스 기반의 아이덴티티 및 접근 관리 솔루션(Identity and Access Management, IAM)으로, 손쉽게 SSO, LDAP, 소셜 로그인 같은 기능을 구현할 수 있도록 도와주는 도구이다.  
> keycloak 설치 방법은 여러 가지가 있지만, 이번 프로젝트에서는 Helm Chart를 사용하여 설치하고자 한다.

## keycloak 관련 용어

용어     | 설명
-------|---------------------------------------------------------------------------------------------
OIDC   | OAuth 가 권한 부여만 다루는 것이라면 OIDC 는 OAuth 를 포함하여 인증과 권한부여를 모두 포함한 것이다.<br>SSO 의 구현을 위한 수단으로 사용된다.
Realm  | 인증, 권한 부여가 적용되는 범위를 나타내는 단위이다.<br>SSO 를 적용한다고 했을때 해당 SSO 가 적용되는 범위는 Realm 단위이다.
Client | 인증, 권한 부여 행위를 대행하도록 맡길 어플리케이션을 나타내는 단위이다.<br>그 단위는 웹사이트 혹은 REST API 를 제공하는 서비스도 될 수 있다. 하나의 Realm 에 n개의 Client 를 생성, 관리할 수 있다.
User   | Client 에 인증을 요청할 사용자를 나타낸다. 하나의 Realm 에는 Realm 에 종속된 n개의 User 를 생성하고 관리할 수 있다.<br>기본적으로 User 는 Username, Email, FirstName, LastName 으로 구성되어 있지만 Custom User Attribute 를 사용하면 사용자가 원하는 속성을 추가할 수 있다.
Role   | User 에게 부여할 권한 내용을 나타낸다.<br>여기에는 Keycloak 의 REST API 를 사용할 권한을 부여할 수 있고 사용자가 정의한 권한을 부여할 수도 있다.

## ⚙️ keycloak 설치

### 1. helm repo 추가

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 2. 버전 확인

```shell
helm search repo keycloak --versions
```

![1](assets\post_imgs\2025-01-17-cicd_project_keycloak\1.png)

### 3. helm chart pull

```shell
helm pull bitnami/keycloak --version 24.4.6
tar -xvf keycloak-24.4.6.tgz
```

### 4. certificate 생성

- 추후 배포될 cicd namespace의 서비스들이 공통으로 사용할 certificate 리소스를 생성한다.

```shell
kubectl create namespace cicd
```

```shell
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: foo-bar-wildcard-cert
  namespace: cicd
spec:
  secretName: foo-bar-wildcard-tls
  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
  commonName: "*.foo.bar"
  dnsNames:
    - "*.foo.bar"
```

```shell
kubectl apply -f certificate.yaml
```

### 5. values.yaml

```yaml
auth:
  adminUser: admin
  adminPassword: # password 지정

---

postgresql: 
  enabled: true # 외부 db가 있다면 false후 externalDB 항목 작성
  auth:
    postgresPassword: # password 지정
    username: bn_keycloak
    password: # password 지정

---

ingress:
  enabled: true
  ingressClassName: "nginx"

---

  extraTls:
    - hosts:
        - "*.foo.bar"
      secretName: "foo-bar-wildcard-tls" # 앞서 만든 certificate

---

production: true # tls 적용 시 proxy=edge로 써야 함
# https://github.com/bitnami/charts/tree/main/bitnami/keycloak#use-with-ingress-offloading-ssl
proxyHeaders: ""
proxy: "edge" # 해당 옵션 미설정 시 web으로 접속하면 too many redirection 에러 발생
```

### 6. helm chart 배포

```shell
helm upgrade -i keycloak ../keycloak -n cicd
```

> 배포 후 postgres가 pending 상태일텐데 pvc가 pv에 바운드 하지 못하여서 발생함.  
> 대응되는 pv를 hostpath로 생성하여 준다.
{: .prompt-danger }

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-keycloak-postgresql-0
  labels:
    app.kubernetes.io/component: primary
    app.kubernetes.io/instance: keycloak
    app.kubernetes.io/name: postgresql
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: /data/keycloak
    type: DirectoryOrCreate
```

> 배포 후 admin으로 접속하면 아래와 같이 임시 admin 계정이기 때문에 영구 admin 계정을 만들어야 한다고 한다.
{: .prompt-info }

![2](assets\post_imgs\2025-01-17-cicd_project_keycloak\2.png)

- 새로운 user 생성

![3](assets\post_imgs\2025-01-17-cicd_project_keycloak\3.png)

- password 설정

![4](assets\post_imgs\2025-01-17-cicd_project_keycloak\4.png)

- admin role 매핑
  - 필터를 role로 변경 후 나오는 role을 모두 체크하여 생성

![5](assets\post_imgs\2025-01-17-cicd_project_keycloak\5.png)

- 이후 기존의 temp admin은 계정을 삭제해도 무방하다.
- 경고 문구가 사라진 모습

![6](assets\post_imgs\2025-01-17-cicd_project_keycloak\6.png)

## 📑 Reference

- <https://devocean.sk.com/blog/techBoardDetail.do?ID=166983>
- <https://honglab.tistory.com/265>
- <https://github.com/bitnami/charts/tree/main/bitnami/keycloak#use-with-ingress-offloading-ssl>
