---
title: CI/CD 구축기 - 4. ingress-controller & metallb 설치
date: 2025-01-14 23:00:00 +09:00
categories: [Development, DevOps]
tags: [Kubernetes, Infra, CICD, DevOps, Nginx, Metallb]
---

## 🛠️ Nginx ingress-controller 설치

### 1. 버전 호환성 확인

- k8s와 호환되는 ingress-controller 버전 확인
  - <https://github.com/kubernetes/ingress-nginx#support-versions-table>
- k8s 1.32와 호환되는 v1.12.0 설치

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml
```

## 🖥️ Metallb 설치

> ingress-controller는 클러스터 내에서 http, https 요청을 라우팅 하는 역할을 한다. 그러나, 클러스터 외부에서 요청을 받을 방법이 없으면 ingress-controller만 있다고 해서 외부에서 접근할 수 없다.  
> 외부에서 ingress-controller로 접근하려면 외부 IP 주소를 할당 받아야 한다. 이 때 LoadBalancer 서비스가 필요한데, 클라우드 환경에서는 보통 LoadBalancer가 자동으로 생성 되지만, 베어메탈 환경에서는 자동으로 외부 IP가 할당되지 않기 때문에, 이를 위해 Metallb를 사용한다.

### 1. Metallb 설치

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml
```

### 2. IP주소 Pool 구성

- metallb는 클러스터 외부에 노출할 ip주소 범위를 설정해야 한다.
- 이번에 구성할 노드는 단일 노드이기 때문에 아래와 같이 구성하였다.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-address-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.2.14-10.0.2.14
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advertisement
  namespace: metallb-system
spec: {}
```

```shell
kubectl apply -f metallb_config.yaml
```

- metallb 설치 전 `Pending`상태인 ingress-controller의 EXTERNAL-IP
![1](assets\post_imgs\2025-01-14-cicd_project_ingress_controller_&_metallb\1.png)
- metallb 설치 후 IP가 할당된 ingress-controller의 EXTERNAL-IP
![2](assets\post_imgs\2025-01-14-cicd_project_ingress_controller_&_metallb\2.png)
