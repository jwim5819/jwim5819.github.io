---
title : helm chart에 포함된 이미지 식별
date : 2024-04-15 23:00:00 +09:00
categories : [Development, Kubernetes]
tags : [Kubernetes, Helm]
---

## 개요

- 폐쇄망 환경에 새로운 helm chart를 배포할 일이 생겨 필요한 docker image를 식별해야하는 일이 생겼다.
- chart가 복잡 할수록 수동으로 식별하기 힘들기 때문에, helm template 명령어를 사용하여 추출하도록 하자.

## 방법

- ```helm template``` 명령어 사용
- 현재 ```values.yaml``` 값을 기반으로 생성될 manifest의 내용에서 ```image:``` 필드의 값을 추출하여 정렬하는 방식이다.

    ```bash
    helm template . | grep image: | sed -e 's/[ ]*image:[ ]*//' -e 's/"//g' | sort -u -> 파일명
    ```

## 예시

### 테스트용 helm chart(gitlab) 준비

- helm chart 저장소 등록

    ```bash
    helm repo add gitlab https://charts.gitlab.io/
    ```

- helm repo 업데이트

    ```bash
    helm repo update
    ```

- helm chart 검색

    ```bash
    helm search repo gitlab
    ```

![1](assets\post_imgs\2024-04-15-k8s_helm_extract\1.png)

- helm chart pull
  - ```helm pull {repo}/{chart} --version {version}```

    ```bash
    helm pull gitlab/gitlab
    ```

### template 추출(차트 위치에서 진행)

- 이미지만 식별

    ```bash
    helm template . | grep image: | sed -e 's/[ ]*image:[ ]*//' -e 's/"//g' | sort -u
    ```

![2](assets\post_imgs\2024-04-15-k8s_helm_extract\2.png)

- manifest 자체를 추출하고 싶을 경우

    ```bash
    helm template . > export.yaml

    # namespace, release name 등을 명령어에 포함시키면 적용되어 추출됨.
    ```
