---
title: Amazon EKS Cluster Endpoint 유형
date: 2025-01-09 12:00:00 +0900
categories: [AWS, Container, Kubernetes]
tags: [Container, Kubernetes, k8s, Amazon EKS, EKS, Endpoint] # TAG names should always be lowercase
---

Amazon EKS 클러스터를 생성할 때 클러스터 엔드포인트 액세스 유형을 지정해야한다.

클러스터 엔드포인트는 컨트롤 플레인의 `쿠버네티스 API 서버 엔드포인트`[^1]를 의미한다.

클러스터 엔드포인트는 시스템 요구사항에 맞춰 유형을 지정하여 생성하는데 이 때 유형별로 어떤 차이점이 있는지와 어떤 아키텍처 구조인지 궁금했다.

우선 클러스터 엔드포인트는 크게 3가지 유형으로 `Public`, `Private`, `Public and Private` 로 나뉜다.

#### 1. Public Endpoint

![Light mode only](/assets/img/posts/2025-01-06-Amazon_EKS_Endpoint_Type/public-endpoint.png){: .light}
![Dark mode only](/assets/img/posts/2025-01-06-Amazon_EKS_Endpoint_Type/public-endpoint-dark.png){: .dark}

사용자는 인터넷에서 클러스터 API 서버에 접근할 수 있다.
VPC 내에서는 데이터 플레인의 서브넷에서 인터넷이 가능하도록 Nat Gateway 구성이 필수이다.

외부 환경에서 클러스터의 DNS 주소를 nslookup으로 조회하면 퍼블릭 아이피 주소가 조회가 된다.
VPC 환경에서도 외부 환경과 동일하게 nslookup으로 조회하면 퍼블릭 아이피 주소가 조회가 된다.

`PUBLIC ACCESS SOURCE CIDR`은 최초 생성 시 `0.0.0.0/0`으로 설정되어 있으며 보안을 위해 자신의 아이피 주소로 설정하는 것이 좋다.
CIDR은 최대 40개까지 추가할 수 있다.

그러나 데이터 플레인에서 컨트롤 플레인의 API 서버와 통신할 때 Amazon 네트워크 내에서만 통신하며 외부 인터넷으로 통신이 되지 않는다.

#### 2. Private Endpoint

![Light mode only](/assets/img/posts/2025-01-06-Amazon_EKS_Endpoint_Type/private-endpoint.png){: .light}
![Dark mode only](/assets/img/posts/2025-01-06-Amazon_EKS_Endpoint_Type/private-endpoint-dark.png){: .dark}

프라이빗 엔드포인트로 설정하면 `EKS OWNED ENI`를 통한 접근만 허용된다.

> `EKS OWNED ENI`는 컨트롤 플레인의 네트워크 인터페이스를 의미한다.

컨트롤 플레인의 API 서버에 접근하기 위해서는 데이터 플레인에서 `EKS OWNED ENI`로 통신이 가능해야한다.

프라이빗 엔드포인트로 설정한 경우 Amazon EKS에서 사용자 대신 Route 53 호스팅 영역에 등록하여 클러스터 엔드포인트를 제공한다.
AWS에 의해 관리되며 사용자 계정에서는 제공된 호스트 영역을 확인할 수 없다.

그리고 사용자 VPC에서 `enableDnsHostnames`와 `enableDnsSupport`를 활성화 해야하며 DHCP 옵션 세트에 도메인 이름 서버 목록에 AmazonProviderDNS가 포함되어 있어야 한다.

외부 환경에서 클러스터의 DNS 주소를 nslookup으로 조히하면 프라이빗 아이피 주소가 조회가 된다.
VPC 환경에서 클러스터의 DNS주소를 nslookup으로 조회하면 프라이빗 아이피 주소가 조회가 된다.
여기서 조회되는 프라이빗 아이피 주소는 `EKS OWNED ENI`의 프라이빗 아이피 주소이다.

VPC 환경에서 인터넷 게이트웨이를 사용하지 않다면 VPC 엔드포인트[^2]를 구성해야 한다.

#### 3. Public and Private Endpoint

![Light mode only](/assets/img/posts/2025-01-06-Amazon_EKS_Endpoint_Type/public-and-private-endpoint.png){: .light}
![Dark mode only](/assets/img/posts/2025-01-06-Amazon_EKS_Endpoint_Type/public-and-private-endpoint-dark.png){: .dark}

퍼블릭&프라이빗 엔드포인트는 위에서 설명한 퍼블릭, 프라이빗 엔드포인트가 통합된 형태로 이해하면 된다.

외부환경에서는 허용된 CIDR을 기반으로 퍼블릭 엔드포인트 주소로 통신되며 VPC 환경에서는 Route53 호스팅 영역을 통해 `EKS OWNED ENI` 아이피 주소로 통신된다.

## Ref.

[^1]: [클러스터 API 서버 엔드포인트에 대한 네트워크 액세스 제어](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/cluster-endpoint.html)
[^2]: [프라이빗 클러스터 구성](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/private-clusters.html)
