---
title: CI/CD 구축기 - 7. Harbor 설치 및 SSO 연동
date: 2025-01-19 23:00:00 +09:00
categories: [Development, DevOps]
tags: [Kubernetes, Infra, DevOps, Harbor, Keycloak, SSO]
---

## 🛠️ Harbor 설치

### 1. helm chart 버전 확인

```shell
helm search repo harbor --versions
```

![1](assets\post_imgs\2025-01-19-cicd_project_harbor\1.png)

### 2. helm chart pull

```shell
helm pull bitnami/harbor --version 24.1.7
tar -xvf harbor-24.1.7.tgz
```

### 3. values.yaml

```yaml
adminPassword: # password

---

externalURL: https://harbor.foo.bar

---

exposureType: ingress
service:
  type: clusterIP

---

ingress:
  core:
    ingressClassName: "nginx"
    hostname: harbor.foo.bar
    extraTls:
    - hosts:
        - "*.foo.bar"
      secretName: "foo-bar-wildcard-tls"
```

### 4. helm chart 배포

```shell
helm upgrade -i harbor ../harbor -n cicd
```

> keycloak postgres와 마찬가지로 pvc에 대응되는 pv가 없어서 pod가 PENDING상태가 될 것이다.
> `localpath provisioner`같은 프로비저닝 도구를 사용해도 되지만, 이번 프로젝트에서는 수동으로 pv를 생성하기로 한다.
{: .prompt-danger }

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-harbor-postgresql
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/data/harbor/postgresql"
    type: DirectoryOrCreate
  storageClassName: ""
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-harbor-trivy
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/data/harbor/trivy"
    type: DirectoryOrCreate
  storageClassName: ""
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-harbor-jobservice
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/data/harbor/jobservice"
    type: DirectoryOrCreate
  storageClassName: ""
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-harbor-registry
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/data/harbor/registry"
    type: DirectoryOrCreate
  storageClassName: ""
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-harbor-redis-master
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/data/harbor/redis-master"
    type: DirectoryOrCreate
  storageClassName: ""
```

## 📑 Reference

- <>
