---
title: CI/CD 구축기 - 7. Harbor 설치 및 SSO 연동
date: 2025-01-19 23:00:00 +09:00
categories: [Development, DevOps]
tags: [Kubernetes, Infra, DevOps, Harbor, Keycloak, SSO, CICD]
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

## 🔐 Keycloak - Harbor SSO 연동

### 1. Keycloak Realm 및 Client 생성

- `Create realm`
![2](assets\post_imgs\2025-01-19-cicd_project_harbor\2.png)

- Realm과 Client 참고
  - <https://jwim5819.github.io/posts/cicd_project_keycloak/>
  
- `Create Client`
![3](assets\post_imgs\2025-01-19-cicd_project_harbor\3.png)
![4](assets\post_imgs\2025-01-19-cicd_project_harbor\4.png)
![5](assets\post_imgs\2025-01-19-cicd_project_harbor\5.png)
![6](assets\post_imgs\2025-01-19-cicd_project_harbor\6.png)

### 2. Harbor configuration

- OIDC `client secret`은 keycloak client의 `credentials` 탭에서 확인 가능하다.
![7](assets\post_imgs\2025-01-19-cicd_project_harbor\7.png)
![8](assets\post_imgs\2025-01-19-cicd_project_harbor\8.png)
- 이후 로그인 화면에 `LOGIN WITH keycloak` 항목이 추가되며, 클릭 시 keycloak 로그인 화면으로 리다이렉팅 된다.
![9](assets\post_imgs\2025-01-19-cicd_project_harbor\9.png)
![10](assets\post_imgs\2025-01-19-cicd_project_harbor\10.png)

## 🐋 Harbor - Containerd 연동

> k8s위에 구축한 Harbor는 private registry이기 때문에, 추가적인 설정이 필요하다.

### 1. private registry dns name 설정

- /etc/hosts 파일에 domain을 추가한다.

```shell
# /etc/hosts

{IP} harbor.foo.bar
```

### 2. containerd configuration

- `/etc/containerd/config.toml` 파일을 수정하여 private registry를 수정해야한다. 만약, 해당 파일이 없으면 생성한다.
- [plugins."io.containerd.grpc.v1.cri".registry] 하위 항목의 mirror 설정에 pricvate registry를 추가한다.

```shell
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."harbor.foo.bar"]
  endpoint = ["https://harbor.foo.bar"]
```

![11](assets\post_imgs\2025-01-19-cicd_project_harbor\11.png)

- [plugins."io.containerd.grpc.v1.cri".registry.configs] 하위 항목에 harbor 접속 id와 pw를 입력 할 수 있다.
  - harbor - keycloak 연동 시 cli에서 사용할 수 있는 secret이 따로 만들어 지게 되는데, 이는 harbor user profile에서 확인 할 수 있다.
- 또한, tls 인증 과정을 skip 할 수 있게 insecure_skip_verify = true 설정을 추가한다.

![12](assets\post_imgs\2025-01-19-cicd_project_harbor\12.png)
![13](assets\post_imgs\2025-01-19-cicd_project_harbor\13.png)

```shell
[plugins."io.containerd.grpc.v1.cri".registry.configs]
  [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.foo.bar".auth]
    username = {id}
    password = {pw}
  [plugins."io.containerd.grpc.v1.cri".registry.configs."iksoon.registry.com".tls]
    insecure_skip_verify = true
```

![14](assets\post_imgs\2025-01-19-cicd_project_harbor\14.png)

### 3. 검증

- harbor에 push해둔 nginx:latest이미지를 crictl pull로 containerd에 가져오기

```shell
sudo crictl pull harbor.foo.bar/library/nginx:latest
```

- containerd의 이미지 삭제 후 deployment 작성으로 k8s 내부에서 이미지 pull 여부 확인

```shell
sudo crictl rmi harbor.foo.bar/library/nginx:latest
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
        image: harbor.foo.bar/library/nginx:latest
        ports:
        - containerPort: 80
```

![15](assets\post_imgs\2025-01-19-cicd_project_harbor\15.png)

## 📑 Reference

- <https://velog.io/@lijahong/0%EB%B6%80%ED%84%B0-%EC%8B%9C%EC%9E%91%ED%95%98%EB%8A%94-Keycloak-%EA%B3%B5%EB%B6%80-OIDC%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-Keycloak-Harbor-SSO-%EC%97%B0%EB%8F%99>
- <https://ikcoo.tistory.com/230>
