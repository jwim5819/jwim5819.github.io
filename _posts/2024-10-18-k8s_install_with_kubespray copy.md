---
title : kubespray를 사용한 k8s cluster 구축
date : 2024-10-18 23:00:00 +09:00
categories : [Development, Kubernetes]
tags : [Kubernetes, Kubespray, Infra]
---

## kubespray란

> **kubespray**란 kubernetes cluster를 쉽게 설치하고 관리할 수 있도록 돕는 오픈소스 프로젝트로 Ansible 기반으로 자동화된 도구이다.
> kubeadm과 같은 kubernetes 표준 설치도구가 있지만, kubespray는 yaml 구성파일을 사용해서 cluster의 설정 및 각종 addon 등을 훨씬 더 직관적으로 설정할 수 있다.
> 개인적으로 사용해본 바로 추가적인 설정을 통해서 kubernetes 설치과정의 모든 의존성, 컨테이너 이미지를 사전에 준비할 수 있어 폐쇄망 환경에서 손쉽게 kubernetes를 설치할 수 있었다.

## 서버 구성

| role          | hostname | cpu | ram |
| ------------- | -------- | --- | --- |
| control plane | k8sm01   | 1   | 4   |
| worker        | k8sw01   | 1   | 4   |
| worker        | k8sw02   | 1   | 4   |

## 준비 과정

> Ansible이 SSH를 통해 다른 하위 노드(k8sw01, k8sw02)에 접속하여 필요한 설치와 구성을 원격으로 자동화하기 때문에, 아래 준비 과정들은 kubespray를 실행할 서버 즉, k8sm01 서버에서만 진행해주면 된다.

- Git 설치
  - 먼저, kubespray 리포지토리를 클론하기 위해 Git을 설치한다.

    ```shell
    sudo apt-get install -y git
    ```

- Kubespray 리포지토리 클론

  ```shell
  git clone https://github.com/kubernetes-sigs/kubespray.git
  ```

- Python 패키지 관리 도구 설치

  ```shell
  sudo apt-get install -y python3-pip
  ```

- 디렉토리 이동

  ```shell
  cd kubespray
  ```

- `pip3`를 사용해 Kubespray에서 요구하는 패키지들을 설치

  ```shell
  sudo pip3 install -r requirements.txt
  ```

- 클러스터 구성을 위해 샘플 인벤토리를 복사하고 설정을 수정한다.

  ```shell
  cp -rfp inventory/sample inventory/mycluster
  ```

- `inventory.ini` 파일을 수정하여 클러스터의 노드 정보를 입력한다.

  ```shell
  vi inventory/mycluster/inventory.ini
  ```

  ```shell
  [all]
  k8sm01 ansible_host=192.168.35.131 ip=192.168.35.131 etcd_member_name=etcd1
  k8sw01 ansible_host=192.168.35.132 ip=192.168.35.132
  k8sw02 ansible_host=192.168.35.133 ip=192.168.35.133

  [kube_control_plane]
  k8sm01

  [etcd]
  k8sm01

  [kube_node]
  k8sw01
  k8sw02

  [calico-rr]

  [k8s-cluster:children]
  kube-master
  kube-node
  calico-rr
  ```

- `k8s-cluster.yml` 파일에서 사용할 쿠버네티스 버전과 컨테이너 런타임을 설정한다.

  ```shell
  vi inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
  ```

  ```yaml
  ...

  ## Change this to use another Kubernetes version, e.g. a current beta release
  kube_version: v1.31.1

  ...

  ## Container runtime
  ## docker for docker, crio for cri-o and containerd for containerd.
  ## Default: containerd
  container_manager: containerd

  ...
  ```

- SSH 키를 생성한 후, 각 노드의 `authorized_keys` 파일에 공개 키를 추가하여 다른 노드에서 접근 가능하도록 한다.

  ```shell
  rm -rf ~/.ssh
  ssh-keygen -t rsa
  cd ~/.ssh
  cat id_rsa.pub >> authorized_keys
  ```

## 설치

- 모든 노드와의 연결상태 확인

  ```shell
  ansible all -m ping -i inventory/mycluster/inventory.ini
  ```

![1](assets\post_imgs\2024-10-18-k8s_install_with_kubespray\1.png)

- `ansible-playbook`을 실행하여 쿠버네티스 클러스터 설치를 시작한다. 이 때, 설치에 root 권한이 필요하기 때문에 `-K` 옵션으로 root 패스워드를 입력 해 줄수 있도록 한다.

  ```shell
  ansible-playbook -i inventory/mycluster/inventory.ini -become --become-user=root cluster.yml -K
  ```

> 설치 과정 중 `RUNNING HANDLER [kubernetes/control-plane : Control plane | wait for ]` task에서 에러가 발생하였는데, reset 후 master node의 cpu와 memory를 각각 2와 8g로 늘려준 후 에러가 사라졌다.
> ansible을 실행하기 위해 일시적이었는지는 모르겠으나, 늘려준 리소스는 계속 유지하여 주었다.
{: .prompt-info }

- 설치완료 후 kubectl 명령어로 node와 pod running상태 확인.

![2](assets\post_imgs\2024-10-18-k8s_install_with_kubespray\2.png)

> 기존에는 k9s라는 터미널 기반의 툴로 cluster를 관리해 왔지만, CKA시험을 위해 `kubectl` 명령어에 익숙해 질 필요가 있기 때문에, 깔지 않았다.