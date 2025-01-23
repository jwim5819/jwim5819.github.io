---
title: CI/CD 구축기 - 2. 프로젝트 구성
date: 2024-12-14 23:00:00 +09:00
categories: [Development, DevOps]
tags: [Kubernetes, Infra, CICD, DevOps]
---

![1](assets\post_imgs\2024-12-14-cicd_project_architecture\1.png)

> **이번 프로젝트의 주요 구성도는 위와 같다.**

## **주요 스택**

1. **Keycloak**:
   - 역할: SSO(Single Sign-On) 인증 제공
   - CI/CD 파이프라인에서 GitLab 및 Harbor와 같은 도구에 통합, 중앙 인증 관리 역할
2. **GitLab**:
   - 역할: 코드 및 매니페스트 관리
   - 소스 코드 및 배포 매니페스트의 중앙 저장소로, 커밋/푸시가 이루어지면 Jenkins를 통해 파이프라인이 실행됨
3. **Jenkins**:
   - 역할: CI 프로세스 관리
   - GitLab의 트리거를 받아 빌드, 테스트, 도커 이미지 생성, 매니페스트 업데이트 등의 작업을 수행
4. **Harbor**:
   - 역할: Docker 이미지 저장소
   - Jenkins에서 생성된 이미지를 저장하고 ArgoCD가 이미지를 Kubernetes 클러스터로 배포할 때 사용
5. **ArgoCD**:
   - 역할: CD 프로세스 관리
   - GitOps 방식으로 배포 매니페스트를 Kubernetes 클러스터에 동기화하고, 자동 및 수동 배포를 지원
6. **Kubernetes**:
   - 역할: 애플리케이션 실행 환경
   - Dev(개발 환경)과 Prod(프로덕션 환경) 두 가지 네임스페이스로 분리하여 각기 다른 배포 전략을 적용
7. **Telegram**:
   - 역할: 알림 시스템
   - Jenkins 및 ArgoCD의 배포 작업 완료 및 배포 상태를 팀에게 실시간으로 알림

## **CI/CD 프로세스**

1. **코드 커밋 및 푸시**
   - GitLab에 소스 코드 푸시, CI 파이프라인 트리거
     - 트리거: GitLab Webhook → Jenkins.
2. **CI: Jenkins를 통한 빌드 및 테스트**
   - Jenkins가 코드를 받아서 다음 작업을 수행
     - 소스 코드 빌드
     - 자동화된 테스트 실행
     - 성공 시 도커 이미지 생성 및 Harbor에 푸시
3. **Docker 이미지 저장**
   - Jenkins가 생성한 이미지는 Harbor에 저장
     - 이미지 태그와 버전 관리
     - SSO(Keycloak) 인증을 통해 접근 제어
4. **매니페스트 업데이트**
   - 새로운 이미지 태그를 포함한 Kubernetes 매니페스트 파일을 업데이트
   - 업데이트된 매니페스트는 GitLab의 매니페스트 저장소에 푸시
5. **CD: ArgoCD를 통한 동기화 및 배포**
   - ArgoCD가 GitLab 매니페스트 저장소를 모니터링하며 변경 사항을 감지
   - Dev 환경에는 자동으로 동기화하여 배포를 진행
   - Prod 환경에는 수동 배포로 안전성을 확보
6. **Kubernetes 클러스터로 배포**
   - Dev와 Prod 환경으로 각각 배포가 진행
   - Dev는 자동 배포, Prod는 매뉴얼 승인이 필요함
