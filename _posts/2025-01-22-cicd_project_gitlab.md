---
title: CI/CD 구축기 - 8. Gitlab 설치 및 설정
date: 2025-01-22 23:00:00 +09:00
categories: [Development, DevOps]
tags: [Kubernetes, Infra, DevOps, Git, Gitlab, Keycloak, SSO, CICD]
---

## 😺 Gitlab 설치 (Helm)

### 1. helm repo 추가

```shell
helm repo add gitlab https://charts.gitlab.io/
helm repo update
```

### 2. 버전 확인

```shell
helm search repo gitlab --versions
```

![1](assets\post_imgs\2025-01-22-cicd_project_gitlab\1.png)

### 3. helm chart pull

```shell
helm pull gitlab/gitlab --version 8.8.1
tar -xvf gitlab-8.8.1.tgz
```

### 4. values.yaml

```yaml
global:
  edition: ce    # comunity edition 사용 설정
  hosts:
    domain: foo.bar    # 기본 도메인 설정 (서브도메인 `gitlab.`과 결합)
  ingress:
    configureCertmanager: false    # Cert-Manager 사용 안 함(기존 tls 사용)
    tls:
      enabled: true    # TLS 활성화
      secretName: foo-bar-wildcard-tls
    omniauth:
      enabled: true    # OmniAuth 기본 활성화
      allowSingleSignOn: ["openid_connect"]
      providers:
        - secret: oidc-provider
          key: gitlab-oidc.json
  gitaly:
    tls:
      enabled: true    # TLS 활성화
      secretName: foo-bar-wildcard-tls
  kas:
    tls:
      enabled: true    # TLS 활성화
      secretName: foo-bar-wildcard-tls
  registry:
    tls:
      enabled: true    # TLS 활성화
      secretName: foo-bar-wildcard-tls

certmanager:    # Cert-Manager 사용 안 함
  installCRDs: false
  install: false
  rbac:
    create: false

gitlab-runner:    # jenkins와 argocd를 활용하여 cicd 프로세스를 구축할 예정이기 때문에 제외
  install: false
```

### 5. oidc provider secret 생성

- json 파일 생성 후 base64 encoding

```json5
{
    "name": "openid_connect",
    "label": "Keycloak",    // 로그인 화면 버튼에 표시 될 텍스트
    "args": {
      "name": "openid_connect",
      "scope": ["openid","profile","email"],
      "response_type": "code",
      "issuer": "https://keycloak.foo.bar/realms/cicd",    // keycloak realm issuer
      "discovery": true,
      "client_auth_method": "query",
      "uid_field": "uid",
      "send_scope_to_token_endpoint": "false",
      "pkce": true,
      "client_options": {
        "identifier": "gitlab",    // keycloak client
        "secret": "jfpxQn4tPS3iMnMr4CUp1re8Y4D6L1Dp",    // keycloak client secret
        "redirect_uri": "https://gitlab.foo.bar/users/auth/openid_connect/callback"    // keycloak realm redirect url
    }
  }
}
```

```shell
cat gitlab-oidc.json | base64 -w 0
```

- oidc 정보를 담은 secret 생성

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: oidc-provider
  namespace: cicd
data:
  gitlab-oidc.json: "base64 encoding value"
```

```shell
kubectl apply -f gitlab-oidc.yaml
```

> 이후 keycloak에서 harbor client 생성때와 같이 gitlab client를 생성한다.
> redirect url은 위의 json 내용 참고
{: .prompt-info }

### 6. helm chart 배포

```shell
helm upgrade -i gitlab -n cicd ../gitlab/
```

> 배포 후 대응되는 pv를 hostpath로 생성.
{: .prompt-info }

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-gitlab-postgresql
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: "/data/gitlab/postgresql"

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-gitlab-minio
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: "/data/gitlab/minio"

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-gitlab-prometheus
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: "/data/gitlab/prometheus"

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-gitlab-redis
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: "/data/gitlab/redis"

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-gitlab-gitaly
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: "/data/gitlab/gitaly"

```

### 7. web service 접속 확인

- gitlab.foo.bar로 접속

![2](assets\post_imgs\2025-01-22-cicd_project_gitlab\2.png)

> default username은 root이고, root passwd는 `gitlab-initial-root-password`의 값을 base64로 decoding 하면 얻을 수 있다.
{: .prompt-info }

- keycloak으로 redirect 할 수도 있다.
  - keycloak 계정은 로그인 후 admin의 approve가 필요함.

![3](assets\post_imgs\2025-01-22-cicd_project_gitlab\3.png)

## 📑 Reference

- <https://docs.gitlab.com/charts/installation/>
