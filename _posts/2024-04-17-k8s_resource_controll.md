---
title : k8s의 자원 관리(QoS)
date : 2024-06-17 23:00:00 +09:00
categories : [Development, Kubernetes]
tags : [Kubernetes, Resource, QoS]
---

## Kubernetres QoS

- 쿠버네티스는 리소스의 과부하로 프로세스에 대한 Kill 이벤트가 발생하는 경우 Kill의 우선순위를 보장하기 위해 QoS(Quality of Service)를 지정한다.
- QoS에는 세 가지 클래스가 있다.
  - `Guaranteed`: 파드 리소스 제약 중 Requests와 Limits가 모든 컨테이너에 존재하며, 값이 일치할 경우. 오버 커밋이 허용되지 않기 때문에 자원 사용을 안정적으로 사용할 수 있다고 봄. 보장 우선 순위가 가장 높음.
  - `Burstable`: limits가 requests보다 클 경우. requests만큼의 리소스를 보장받지만, 상황에 따라서는 limits까지 사용 가능하다. `Guaranteed`와 `BestEffort`에 속하지 않는 파드. 보장 우선 순위 중간.
  - `BestEffort`: 파드 리소스 제약을 사용하지 않을 경우. 노드에 유휴 자원이 존재한다면, 제한 없이 모든 자원을 사용한다. 보장 우선 순위가 가장 낮음.
  - 하지만 이 순위가 항상 절대적이지는 않는데, 설명한 것 처럼 우선순위는 기본적으로 `Guaranteed` > `Burstable` > `BestEffort`지만 `Burstable` 과 `BestEffort`클래스의 파드들은 **현재 메모리를 얼마나 사용하고 있는지**에 따라 우선순위가 역전 될 수 있다.

## 메모리 관리의 중요성

- 쿠버네티스에는 Pod의 cpu와 memory 리소스를 제한할 수 있다. 만약 cpu가 node의 가용한 100%를 사용하고 있다면, 조금 느려지는 것 외에 별다른 문제는 발생하지 않는다.
- 그런데 memory의 경우는 조금 다르다. 파드의 memory가 limit을 넘는다면 파드는 OOMKill이 될 것이고, 노드의 가용한 전체 메모리가 초과된다면 노드 자체가 마비될 수도 있다.
- 그렇다면, 쿠버네티스는 memory 부족 현상을 어떤 식으로 처리할까?

## 메모리 확보 프로세스

- 만약 node에 MemoryPressure가 발생 할 경우(기본적으로 100Mi이하일 때 해당 이벤트가 발생하도록 kubelet에 설정되어 있음.) QoS 순위에서 보장 순서가 낮은 파드를 다른 노드로 방출(Eviction)시키게 된다.
- 만약 kubelet이 eviction을 수행하기도 전에 갑작스럽게 메모리 사용량이 급증하면 OOMkill이 발생한다.
  - 파드의 limit을 넘어서는 메모리 사용과, 위의 경우. 둘 다 OOMKilled status를 가지게 된다.

## Eviction 시작 조건

![1](assets\post_imgs\2024-04-17-k8s_resource_controll\1.png)

- 쿠버네티스는 현재 상태(memory.available)를 eviction signal로 판단한다. eviction signal이  eviction threshold에 설정된 값보다 낮아졌을 때(즉, allocatable한 영역을 memory사용량이 넘어서서 memory.available이  eviction threshold 보다 낮을 때) eviction이 시작 된다.
- eviction signal은  monitoring interval 마다 실행되는데 특별히 설정을 하지 않았다면 default 10초 주기로 확인한다.

## Eviction 방식

- eviction 프로세스에는 두가지 방식이 있다. 만약 특정 파드의 메모리 사용량이 일시적으로 증가했다가, 다시 안정화가 되는 경우 순간적인 peak때문에 동작중인 파드를 죽이는 것은 비효율 적일수 있다. 따라서 쿠버네티스는 soft eviction과 hard evition을 지원하는데, 차이점은 다음과 같다.
  - soft eviction: eviction이 발생하기 까지 유예 기간을 준다.
  - hard eviction: eviction threshold를 만나는 즉시 eviction이 이루어 진다.
- 참고로, kubelet설정중에 `eviction-pressure-transition-period`라는 값이 있는데, 이는 node의 status를 변경하는 시간이다. 만약 soft eviction이 일어나 eviction을 시킬지 말지 유예기간을 주고있는 파드가 memory pressure 기준 위, 아래로 변동한다면 node의 status가 자주 변동될 수 있다. 이는 scheduling 성능에 영향을 미치기 때문에, node의 status 변경에 유예기간을 주는 것이다.

## 테스트 환경

- 테스트를 진행할 노드의 상태는 다음과 같다.
![2](assets\post_imgs\2024-04-17-k8s_resource_controll\2.png)
![3](assets\post_imgs\2024-04-17-k8s_resource_controll\3.png)
- 총 메모리는 약 16Gi 정도이며, kube-reserved, system-reserved memory는 각 1.5Gi, eviction threshold는 memory.available < 1Gi로 설정하였다.
![4](assets\post_imgs\2024-04-17-k8s_resource_controll\4.png)
- allocatable한 메모리는 약 12Gi 이다.

## 테스트

- **case. 1**
  - 테스트를 위해 메모리에 스트레스를 줄 수 있는 파드를 배포한다.
  - limits가 3000Mi이고 실제 부하되는 메모리는 5000Mi 이므로, OOMKill이 발생하여야 한다.
    - 참고로 args들은 다음을 뜻한다.
      - `--vm`: 생성할 작업자 수(1 당 1000m의 cpu를 사용한다.)
      - `--vm-bytes`: 작업자에게 주어진 바이트(memory)
      - `--vm-hang`: 메모리 액세스를 중지할 초라고 하는데 잘 모르겠다.
      - `--vm-keep`: 메모리 사용률을 일관되게 유지

        ```yaml
        # stress-01.yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: stress-01
          namespace: stress
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: stress-01
          template:
            metadata:
              labels:
                app: stress-01
            spec:
              containers:
                - name: stress-01
                  image: polinux/stress
                  resources:
                    requests:
                      memory: "2000Mi"
                    limits:
                      memory: "3000Mi"
                  command: ["stress"]
                  args: ["--vm", "1", "--vm-bytes", "5000M", "--vm-hang", "1", "--vm-keep"]
        ```

    - 배포

        ```bash
        kubectl apply -f stress-01.yaml
        ```

    - 예상대로 OOM kill이 발생하였다.
        ![5](assets\post_imgs\2024-04-17-k8s_resource_controll\5.png)

- **case. 2**
  - 각각 2Gi의 메모리를 점유하는 파드들을 배포하였다.
    - stress-01은 request, limit를 모두 설정해주었다.즉, QoS `Guaranteed` 클래스
    - stress-02는 request만 설정하였다. QoS `Burstable` 클래스
    - stress-03은 resource를 제한하지 않았다. QoS `BestEffort` 클래스
      ![6](assets\post_imgs\2024-04-17-k8s_resource_controll\6.png)
  - kubectl top nodes 명령어로 확인 한 결과 현재 노드의 메모리 점유량은 약 70% 정도이다.
      ![7](assets\post_imgs\2024-04-17-k8s_resource_controll\7.png)
  - 이 때 파드의 메모리 사용량을 높여서 memory pressure를 발생시킨다면 어떻게 될까 ?
  - stress-01의 메모리 사용량을 5Gi로 높이고 request와 limit또한 6Gi로 상향 조정해본다. 예상대로라면 stress-01의 메모리 사용량이 가장 높지만, `Guaranteed` 클래스이기 때문에, 3개의 파드중 보장 우선순위가 가장 낮은 stress-03 파드가 eviction 될 것이다.
  - stress-01의 메모리 사용량을 높이자 보장우선순위가 가장 낮은 stress-03이 pending 상태로 변하였다.
      ![8](assets\post_imgs\2024-04-17-k8s_resource_controll\8.png)
  - event에서는, node가 eviction threshold를 만났고, stress-03이 evicted된 것을 확인 할 수 있다.
      ![9](assets\post_imgs\2024-04-17-k8s_resource_controll\9.png)
  - node의 describe에서는, memory pressure로 인하여 node의 taint가 no scheduling 상태로 설정 된 것을 확인할 수 있다.
      ![10](assets\post_imgs\2024-04-17-k8s_resource_controll\10.png)
  - node의 taint를 강제로 제거하여 다시 스케줄링이 되도록 하였음에도, stress-03은 eviction 된 상태로, 해당 노드에 스케줄링이 되지 않는다.
      ![11](assets\post_imgs\2024-04-17-k8s_resource_controll\11.png)