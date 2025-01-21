---
title: CI/CD 구축기 - 3. k8s & CNI 설치
date: 2025-01-12 23:00:00 +09:00
categories: [Development, DevOps]
tags: [Kubernetes, Infra, CICD, DevOps, CNI]
---

| Component | Version                | Note |
| --------- | ---------------------- | ---- |
| OS        | ubuntu22.04 lts server |      |
| kubeadm   | v1.32.1-1.1            |      |
| kubelet   | v1.32.1-1.1            |      |
| kubectl   | v1.32.1-1.1            |      |

## 🛠️ 설치 전 작업

### 1. 필수 패키지 설치

```shell
sudo apt update
sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
```

### 2. swap 메모리 비활성화

```shell
sudo swapoff -a && sudo sed -i '/swap/s/^/#/' /etc/fstab
```

### 3. 브릿지 네트워크의 iptable 규칙 활성화

- Kubernetes에서 브릿지 네트워크를 통과하는 트래픽도 IP 테이블 규칙을 적용해 네트워크 플러그인이 제대로 동작하도록 한다.

```shell
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

### 4. 방화벽 및 포트확인

- master node

  | **프로토콜** | **방향** | **포트범위** | **목적**                | **used by**          |
  | ------------ | -------- | ------------ | ----------------------- | -------------------- |
  | TCP          | inbound  | 6443         | kubernetes API server   | ALL                  |
  | TCP          | inbound  | 2379 ~ 2380  | etcd server client API  | kube-apiserver, etcd |
  | TCP          | inbound  | 10250        | kubelet API             | self, control plane  |
  | TCP          | inbound  | 10251        | kube-scheduler          | self                 |
  | TCP          | inbound  | 10252        | kube-controller-manager | self                 |

- worker node

  | **프로토콜** | **방향** | **포트범위**  | **목적**                                  | **used by** |
  | ------------ | -------- | ------------- | ----------------------------------------- | ----------- |
  | TCP          | inbound  | 10250         | kubelet API                               | ALL         |
  | TCP          | inbound  | 30000 ~ 32767 | Default port range for NodePort Services. | ALL         |

> 나는 개인 프로젝트 이기 때문에 그냥 방화벽 자체를 꺼버렸지만 상황에 맞게 방화벽을 조절하는 것이 좋다.
{: .prompt-info }

- 방화벽 헤제

```shell
sudo ufw disable
sudo ufw status
```

## 🖥️ kubeadm, kubectl, kubelet 설치

### 1. kubernetes 패키지 저장소의 공개 서명 키 다운로드

```shell
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### 2. kubernetes 패키지 저장소 추가

```shell
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
# 해당하는 버전만 있는 레포지터리이기 때문에 사용하고자 하는 버전으로 수정 필요.
```

### 3. 설치 가능한 버전 확인

```shell
sudo apt-cache madison kubeadm
sudo apt-cache madison kubelet
sudo apt-cache madison kubectl
```

### 4. 설치 후 버전 고정

```shell
sudo apt-get update
sudo apt-get install -y kubelet='1.32.1-1.1'
sudo apt-get install -y kubeadm='1.32.1-1.1'
sudo apt-get install -y kubectl='1.32.1-1.1'
sudo apt-mark hold kubelet kubeadm kubectl
```

### 5. 설치 확인

```shell
kubeadm version
kubectl version
kubelet --version
```

## 🛳️CRI(Container Runtime Interface)설치

### 1. docker gpg키 추가

```shell
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### 2. docker 저장소 추가

```shell
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```

### 3. containerd 설치

```shell
sudo apt-get install -y containerd.io
```

### 4. containerd 서비스 등록 및 상태 확인

```shell
sudo systemctl enable containerd --now
sudo systemctl status containerd
```

## 📝 k8s cluster init

### 1. kubeadm init

```shell
sudo kubeadm init --pod-network-cidr 192.168.0.0/16
```

> pod cidr 설정은 다음에 유의하여 설계한다.
>
> 1. 충돌 방지: 사내 네트워크나 다른 클러스터의 대역과 겹치지 않도록 설정해야 한다.
> 2. IP확보: 클러스터 규모와 pod수에 맞는 대역을 설정해야 하며, 사용할 CNI 플러그인 요구 사항도 고려한다.
{: .prompt-info }

### 2. 사용자 환경에 kubectl 명령어 접근 설정

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3. master node taint 해제

- init이 완료되면 join token을 주지만, 이번 프로젝트는 단일 노드로 구성 할 예정이기에 생략하고 master node의 taint를 해제한다.

```shell
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-
```

## 🌎 CNI 설치 (calico) 및 k9s 설치

### 1. Calico 설치

```shell
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### 2. k9s 설치

- 클러스터 관리 편의성을 위해 k9s을 설치한다.
- 최신버전 확인
  - <https://github.com/derailed/k9s/releases>

```shell
K9S_VERSION=v0.32.7
curl -sL https://github.com/derailed/k9s/releases/download/${K9S_VERSION}/k9s_Linux_amd64.tar.gz | sudo tar xfz - -C /usr/local/bin k9s
```

## ⚠️ 트러블 슈팅

> **unknown service runtime.v1alpha2.RuntimeService**
>
> - 컨테이너 런타임에서 CRI 기능을 비활성화 한 경우이므로 아래 작업을 진행
{: .prompt-danger }

```shell
sudo vi /etc/containerd/config.toml
# disabled_plugins = ["cri"] 줄을 주석 처리
sudo systemctl restart containerd.service
```

> **ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1**
>
> - 에러 메세지 그대로 /proc/sys/net/ipv4/ip_forward값을 1로 설정하라는 메세지.
> - /proc/sys/net/ipv4/ip_forward는 패킷을 포워딩하는 옵션정보가 담겨있는 파일인데 0으로 설정되어 있으면 비활성화, 1로 설정되어 있으면 활성화된 것이다.
{: .prompt-danger }

```shell
sudo vi /etc/sysctl.conf
# net.ipv4.ip_forward=1행의 주석을 제거
# 이후 변경된 설정 적용을 위해 아래 명령어 실행
sudo sysctl -p
```

> **kube-system pod들이 자꾸 죽을 때**
>
> - 설치 이후 kube-system namespace의 pod들이 자꾸 죽으며, node가 notReady 상태로 변함.
> - containerd cgroup 관리 방식을 systemd로 수정해준다.
> - 이 설정은 systemd와 호환되는 방식으로 컨테이너의 cgroup을 관리하게 한다.
{: .prompt-danger }

```shell
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

## 📑 Reference

- <https://kubernetes.io/ko/docs/reference/networking/ports-and-protocols>
- <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>
- <https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/>
