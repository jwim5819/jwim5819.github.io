---
title: Docker insecure-registry 추가 시 docker start fail
date: 2024-04-06 23:00:00 +09:00
categories: [Development, Infra]
tags: [Docker]
---

> 평소와 같이 /etc/docker/daemon.json에 insecure-registry를 추가 후 docker를 재기동 하였더니, restart가 되지 않는 현상이 발생하였다...

## daemon.json 수정

- `sudo vi /etc/docker/daemon.json`

  ```bash
  {
      "insecure-registries" : ["{domain}"]
  }
  ```

- 재기동

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl restart docker
  ```

- 에러 발생

![1](assets\post_imgs\2024-04-06-docker_insecure_reg\1.png)

## 원인

- 현재 서버에 설치된 docker는 kubespray설치과정에서 자동으로 설치된 docker인데, kubespary의 insecure-registy 옵션이 docker service daemon의 설정에 이미 insecure-registry를 추가하여 옵션 충돌이 일어났음.

- `/etc/systemd/system/docker.service.d/docker-options.conf`

![2](assets\post_imgs\2024-04-06-docker_insecure_reg\2.png)

## 해결조치

- `docker-options.conf`과 `daemon.json`간에 중복되는 옵션을 제거하면 해결 가능.
