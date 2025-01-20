---
title : k8s에 배포된 object에서 manifest 추출
date : 2024-04-06 24:00:00 +09:00
categories : [Development, Kubernetes]
tags : [Kubernetes, Deploy]
---

## 개요

- kubernetes에 배포되어있는 object를 yaml형태의 manifest로 추출한다.
- 배포 된 상태 그대로 가져오기 때문에, ```resourceVersion```, ```uid```, ```status``` 등 잡다한 내용도 같이 추출되어 그렇게 보기 좋지는 않지만, 재 배포에 문제는 없다.
- 추후 더 스마트한 방법이 있다면 내용 추가하겠음.

## 방법

- ```kubectl get {object 종류} {object 이름} -n {namespace} -o yaml > {파일명}.yaml```

    ```bash
    kubectl get deployments.apps nginx -n nginx -o yaml > nginx_deploy.yaml
    ```
