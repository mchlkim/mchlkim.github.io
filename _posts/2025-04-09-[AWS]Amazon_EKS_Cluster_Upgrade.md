---
title: Amazon EKS Cluster Kubernetes 업그레이드
date: 2025-04-09 12:00:00 +0900
categories: [AWS, Container, EKS]
tags: [Container, Kubernetes, k8s, Amazon EKS, EKS, Upgrade]
---

## 개요

Kubernetes는 평균 4개월에 한 번씩 새로운 마이너 버전을 릴리스[^1]하고 Amazon EKS는 이러한 마이너 버전의 업스트림 릴리스 및 지원 중단 주기를 따른다.</br>
이로 인해 마이너 버전 릴리스 후 14개월 동안 Amazon EKS의 표준 지원이 제공되고 이후 12개월 동안은 확장 지원으로 전환된다.</br>
확장 지원 기간에는 클러스터 시간당 추가 비용을 지불해야 한다.</br>
물론 확장 지원 여부를 설정할 수는 있지만, 기본적으로 활성화되어 있으며 확장 지원도 12개월 동안만 지원되기 때문에 업그레이드 작업은 피할 수 없다.

---

### 업그레이드 고려사항

1. 버전 차이 정책[^2]
2. Kubernetes 변경 내역[^3]
3. 컨트롤 플레인 로깅 활성화[^4]
4. Amazon EKS 변경 내역[^5]
5. Amazon EKS 추가 기능[^6]
6. Amazon EKS 네트워크[^7][^8]

---

#### 1. 버전 차이(skew) 정책

Kubernetes에서는 구성 요소 간에 지원되는 최대 버전 차이가 존재한다.
버전은 `x.y.z`로 표현 된다.
|x|y|z|
|:-----|:-----|-----:|
|메이저 버전|마이너 버전|패치 버전|

#### 지원되는 버전 차이 (1.32 기준)

##### kube-apiserver

지원하는 버전: 1.32, 1.31

##### kubelet

> `kublet`은 `kube-apiserver`보다 최신 버전일 수 없음.

지원하는 버전: 1.32, 1.31, 1.30

##### kube-proxy

> `kube-proxy`는 `kube-apiserver`, `kubelet`보다 최신 버전일 수 없음.

지원하는 버전: 1.32, 1.31, 1.30, 1.29

##### controller-manager, scheduler, cloud-controller-manager

지원하는 버전: 1.32, 1.31

##### kubectl

> `kubectl`의 경우 바로 윗단계의 상위 버전도 지원된다.

지원하는 버전: 1.33, 1.32, 1.31

##### 데이터 플레인

> `데이터 플레인`은 -3 마이너 버전까지 지원하지만 권장되지는 않는다.

지원하는 버전: 1.32, 1.31, 1.30, 1.29

#### 2. Kubernetes 변경 내역

어떤 변경 사항이 있는지 확인하여 영향이 있는 리소스를 찾아 변경 사항을 반영이 필요하다.

변경 내역에

#### 3. 컨트롤 플레인 로깅 활성화

Amazon EKS를 사용하는 경우 컨트럴 플레인은 AWS 관리 영역이지만 로그 설정은 우리의 영역이기 때문에 로깅은 별도로 활성화를 해야한다.

#### 4. Amazon EKS 변경 내역

Amazon EKS에서도 Kubernetes에서 영향이 큰 변경 사항도 알려주지만 Amazon EKS와 연관된 내용들도 확인이 필요하다.

#### 5. Amazon EKS 추가 기능

AWS에서 제공하는 추가 기능에서는 대부분 업데이트 버전 정책은 존재하지 않지만 VPC CNI 플러그인의 경우 업데이트 버전 정책이 존재한다.

##### VPC CNI 플러그인

업데이트 시 마이너 버전 기준으로 순차적인 업데이트를 해야한다.
`1.28.x-eksbuild.y`에서 `1.30.x-eksbuild.y`으로 업데이트를 하려면 중간에 `1.29.x-eksbuild.y`로 업데이트를 하고 `1.30.x-eksbuild.y`로 업데이트 해야한다.

#### 6. Amazon EKS 네트워크

1. 서브넷에서는 사용 가능한 아이피 주소가 5개 이상 존재해야한다.
2. VPC 내부에서 퍼블릭 인터넷 통신이 가능한 경우 인터넷 게이트웨이, NAT 게이트웨이, 라우팅, 보안 그룹을 확인하여 통신이 가능한지 확인이 필요하다.
3. VPC 내부에서 퍼블릭 인터넷 통신이 불가능한 경우 각 서비스 별 VPC 엔드포인트, 라우팅, 보안 그룹을 확인하여 통신이 가능한지 확인이 필요하다.

---

## Ref.

[^1]: [Amazon EKS Kubernetes 릴리스 일정](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/kubernetes-versions.html#kubernetes-release-calendar)
[^2]: [Kubernetes 버전 차이 정책](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG)
[^3]: [Kubernetes 변경 내역](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG)
[^4]: [Amazon EKS 컨트럴 플레인 로깅](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/control-plane-logs.html)
[^5]: [Amazon EKS 변경 내역](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/kubernetes-versions-standard.html)
[^6]: [Amazon EKS 추가 기능](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/eks-add-ons.html)
[^7]: [Amazon EKS 네트워킹 요구 사항](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/network-reqs.html)
[^8]: [Amazon EKS 프라이빗 VPC 요구 사항](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/private-clusters.html)
