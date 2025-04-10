---
title: kubectl-ai
date: 2025-04-10 12:00:00 +0900
categories: [Container, kubernetes]
tags: [Container, Kubernetes, k8s, kubectl-ai]
---

## 개요

`kubectl-ai`는 Kubernetes 클러스터 관리를 위한 CLI를 개선하기 위해 설계된 AI 기반 어시스턴트이다.
현재 지원하는 모델은 Google Gemini만 지원하며, 복잡한 Kubenetes 작업을 자연어를 통해 수행할 수 있도록 도와준다.

---

## 주요 기능
1. 자연어 쿼리 지원
2. 대화형 셸 환경 제공
3. 매니페스트 생성 및 수정 자동화
4. 파이프 입출력 지원

---
## 설치

아래 사전 조건을 만족해야 실행이 가능하다.

### 사전 조건
- Go 1.23 이후 버전
- kubectl 설치 및 구성
- Gemini API Key 

### 다운로드 및 설치
**다운로드 및 설치**
```shell
git clone https://github.com/GoogleCloudPlatform/kubectl-ai.git
cd kubectl-ai
dev/tasks/install
```

**PATH 추가**
```shell
# PATH 추가
echo 'export PATH="$HOME/go/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```
**설치 확인**
```shell
kubectl-ai --help
```

**API Key 등록**
```shell
export GEMINI_API_KEY=<api_key>
```
---
### 사용 방법

#### 사용 방법 1
```shell
kubectl-ai "자연어 쿼리"
```
#### 사용 방법 2
```shell
kubectl-ai < query.txt
# OR
echo "default 네임스페이스의 파드 리스트 조회" | kubectl-ai 
```

#### 사용 방법 3
```shell
cat error.log | kubectl-ai "오류를 설명해줘"
```

#### 사용 방법 4
Interactive Shell로 대화형으로 사용할 수 있다.
```shell
kubectl-ai
```


### 사용 예제
#### 예제 1.클러스터에서 실행 중인 전체 디플로이먼트의 정보 조회

![img.png](/assets/img/posts/2025-04-10-kubectl-ai/example01.png)

#### 예제 2: 현재 실행 중인 워크로드에서 개선사항 확인 
![img.png](/assets/img/posts/2025-04-10-kubectl-ai/example02.png)

#### 예제 3: nginx 파드 생성
![img.png](/assets/img/posts/2025-04-10-kubectl-ai/example03.png)

#### 예제 4: nginx 파드 삭제
![img.png](/assets/img/posts/2025-04-10-kubectl-ai/example04.png)

#### 예제 5: Secret 값 디코딩된 정보 조회 
![img.png](/assets/img/posts/2025-04-10-kubectl-ai/example05.png)

### 사용 후기
자연어를 통해 내가 요청하는 내용을 정확히 이해하고 실행까지 해준다.
터미널과 Gemini를 번갈아가며 질문할 필요 없이, 터미널에서 모든 요청이 처리되기 때문에 굉장히 편리하다.

예제 2번처럼 현재 구성된 워크로드의 문제점 및 개선사항을 알려주고, 다음 단계까지 생각해서 제안해준다.
이걸 잘 응용한다면, 현재 구성된 워크로드나 장애 상황에서 빠르게 문제를 해결할 수 있을 것으로 보인다.

하지만 아직은 출시 초기 단계이기 때문에 몇 가지 문제점도 보인다.
1.	사용자 수락 없이 명령어 실행
민감한 명령어도 사용자 동의 없이 실행되기 때문에 위험할 수 있다.
다만, 사용자 허락을 받고 명령어를 실행해줘 같은 프롬프트를 함께 작성하면 바로 실행되지는 않는다.
2.	민감한 정보 마스킹 또는 제거 미지원
Secret 정보나 로그에 포함된 민감한 정보가 마스킹되거나 제거되지 않는다.
3. 
이 부분은 아직 출시 초기라 그런 듯하고, 앞으로 개선될 것으로 보인다.



