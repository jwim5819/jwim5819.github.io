---
title: CI/CD 구축기 - 5. cert-manager 설치 및 검증
date: 2025-01-17 23:00:00 +09:00
categories: [Development, DevOps]
tags: [Kubernetes, Infra, DevOps, Nginx, Cert]
---

## 🔒 서론

> Kubernetes 환경에서 보안은 매우 중요한 요소다. 특히, 애플리케이션 간의 통신을 안전하게 보호하기 위해서는 TLS(Transport Layer Security) 인증서 관리가 필수적이다. 하지만 인증서를 수동으로 관리하고 갱신하는 일은 번거롭고 실수할 위험이 있다. 이때 cert-manager가 유용하게 사용된다.  
> cert-manager는 Kubernetes 클러스터에서 자동으로 인증서를 발급하고 갱신하는 오픈소스 도구로, 보안을 유지하면서도 효율적인 인증서 관리를 가능하게 한다. 이번 포스팅 에서는 cert-manager를 설치하고, 정상적으로 동작하는지 검증하는 방법에 대해 다룬다.

## ⬇️ Helm 설치

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## ⚙️ cert-manager 설치 및 설정

### 1. helm repo 추가

```shell
helm repo add jetstack https://charts.jetstack.io --force-update
helm search repo jetstack
```

![1](assets\post_imgs\2025-01-17-cicd_project_cert_manager\1.png)

### 2. helm chart 설치

```shell
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.16.3 \
  --set crds.enabled=true
```

### 3. clusterissuer 또는 issuer 생성

- cert-manager에서 인증서를 발급하기 위한 리소스로, 적용 범위에 차이가 있다.
  - Issuer: 네임스페이스에 제한된 인증서 발급 리소스로, 특정 네임스페이스에서만 사용된다.
  - ClusterIssuer: 클러스터 전체에서 사용할 수 있는 인증서 발급 리소스로, 모든 네임스페이스에서 인증서를 발급할 수 있다.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

- 상태 확인
![2](assets\post_imgs\2025-01-17-cicd_project_cert_manager\2.png)

> 다른 예제 처럼 let's encrypt를 사용하여 구축해보고 싶었지만, cert 생성 과정에서 `DNS problem: NXDOMAIN looking up A for nginx.foo.bar`에러가 발생하였다.  
> Let’s Encrypt는 도메인 이름에 대한 DNS 레코드를 검증하는 방식으로 SSL 인증서를 발급하기 때문에, 도메인이 공개 DNS 서버에 등록되지 않으면 인증서를 발급할 수 없다. 따라서, 사설 인증서를 사용하는 방법으로 변경하였다.
{: .prompt-info }

### 4. certificate 리소스 생성

- 이번 프로젝트에서 도메인은 {app}.foo.bar로 통일할 예정이기 때문에, 인증서는 *.foo.bar의 와일드 카드 인증서로 생성하였다.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: foo-bar-wildcard-cert
  namespace: default
spec:
  secretName: foo-bar-wildcard-tls
  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
  commonName: "*.foo.bar"
  dnsNames:
    - "*.foo.bar"
```

- 상태 확인
![3](assets\post_imgs\2025-01-17-cicd_project_cert_manager\3.png)

## ✅ 설치 검증

### 1. 테스트용 nginx pod 배포

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
    - port: 80
  selector:
    app: nginx
```

### 2. ingress 배포

- tls에 앞서 생성한 certificate를 적용

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.foo.bar
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
  tls:
  - hosts:
    - "*.foo.bar"
    secretName: foo-bar-wildcard-tls
```

### 3. Web에서 확인

- 사설인증서를 사용하였기 때문에 최초 접속 시 `신뢰할 수 없는 인증서`라고 나옴.
- 신뢰할 수 있는 인증서로 등록 후 유효한 인증서 확인
![4](assets\post_imgs\2025-01-17-cicd_project_cert_manager\4.png)
