---
title: Docker Hub Rate Limit
date: 2025-04-11 12:00:00 +0900
categories: [Infra, Container, Docker]
tags: [Container, Kubernetes, k8s, docker hub,rate limit]
---

## 개요
Docker 환경 또는 Kubernetes 환경에서 배포를 위해 이미지 빌드 할 때 Docker Hub에서 이미지를 가져와서 사용하는 경우가 많다.  
Docker Hub에서는 Pull에 대한 제한 정책[^1]이 존재한다.  
이로인해 제한량까지 요청하게 된다면 서버에서는 거부를 하게 되며 빌드 단계에서 이미지를 제대로 불러오지 못하기 때문에 오류가 발생한다.  

| 사용자 유형  | Pull 제한  |
|:-------:|:--------:|
| 익명 사용자  | 6시간당 100 |
| 로그인 사용자 | 6시간당 200 |
| 유료 사용자  |  제한 없음   |

Docker Hub에 의존하게 된다면 유료로 사용하지 않으면 제한이 걸리게 된다.  
우회하거나 다른 방안들을 소개하겠다.

---
## 방안
1. Proxy 또는 NAT 사용   
장점이 없는 방법으로 생각된다.
2. 사설 이미지 저장소  
사설 이미지 저장소는 서버에 구축하는 방식, Amazon ECR와 같이 서버리스 방식이 있다.  
서버를 사용하게 되면 유지관리 오버헤드가 늘어나고 서버리스를 사용하면 유지관리 오버헤드는 없다.  
그러나 두 방법 모두 이미지 용량 만큼 비용이 발생하며, 필요한 이미지가 릴리즈되었을 때 사설 저장소에 이미지를 푸시를 해야하는 번거로움이 존재하다.
3. 다른 퍼블릭 이미지 저장소  
GitHub에서 제공하는 GHCR, Red Hat에서 제공하는 Quay.io가 있지만 Amazon ECR Public Galley[^2]가 Docker Hub와 가장 유사하고 편리하다.  

---
## 결정
Amazon ECR Pulbic Gallery의 경우 Pull에 대한 제한 정책[^3]은 Docker Hub보다 더 유연하기 때문에 결정했다.  
단, 월 500GB 데이터까지만 지원, 익명 사용자의 경우 500GB 초과 사용 불가능  
로그인 사용자의 경우 500GB를 모두 사용하면 해당 계정으로 데이터 전송 비용이 청구된다.  

| 사용자 유형  | Pull 제한 |
|:-------:|:-------:|
| 익명 사용자  | 분당 300  |
| 로그인 사용자 | 분당 5000 |

익명 사용자의 경우 분당 300번까지 Pull을 할 수 있기 때문에 제한에 걸리는 일은 없을 것이다.  

#### 로그인 방법
권한이 있는 IAM User Credential 또는 IAM Role에서 사용이 가능하며 IAM User인 경우 해당 계정으로 스위칭이 필요하다.  
IAM Role의 경우 AWS 서비스에 할당되어 있으면 스위칭이 불필요하다.  

```shell
aws ecr-public get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin public.ecr.aws
```

---
## Ref.
[^1]: [Docker Hub usage and limits](https://docs.docker.com/docker-hub/usage/)
[^2]: [Amazon ECR Public Gallery](https://gallery.ecr.aws/)
[^3]: [Amazon ECR Public service quotas](https://docs.aws.amazon.com/ko_kr/AmazonECR/latest/public/public-service-quotas.html)
