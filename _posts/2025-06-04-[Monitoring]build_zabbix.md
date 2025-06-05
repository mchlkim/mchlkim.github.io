---
title: 사내 자빅스 새로 구축하기
date: 2025-06-04 12:00:00 +0900
categories: [Infra, Monitoring]
tags: [Zabbix, Monitoring]
---

## 개요
자빅스는 서버, 네트워크, 애플리케이션 등 다양한 호스트로부터 데이터를 수집할 수 있는 오픈 소스 통합 모니터링 솔루션이다.

팀에서 이미 자빅스를 사용하고 있었으나 담당자의 부재, 인수인계의 부재, 문서의 부재, 데이터베이스 문제로 인해 신규로 구축하는 것으로 결정했었다.  

---

## 설계
기본 자빅스 서버 구조는 프론트엔드와 백엔드가 단일 서버로 구성되었고 데이터베이스도 단일 RDS 인스턴스로 구성되었었다.  
  

기존 자빅스의 문제점은 아래와 같았다.
1. 백엔드에서 모니터링 데이터 수집으로 인한 빈번한 서버 부하 문제
2. 백엔드에서 부하 발생 시 프론트엔드 화면 로딩 문제
3. 데이터베이스 장애로 재부팅 시 긴 다운타임 발생
4. 아이피 주소 기반 통신

위의 문제점을 고려하여 설계를 진행했다.

1. 백엔드, 프론트엔드, 프록시 서버 분리 구성
2. 데이터베이스 이중화
3. 도메인 활용

|  서버   |          역할          | 
|:-----:|:--------------------:|
| 프론트엔드 |      GUI 환경 제공       |
|  백엔드  |      모든 데이터 처리       |
|  프록시  | 원격 또는 분산 환경에서 데이터 수집 |


### 아키텍처 초안 
![draft](/assets/img/posts/2025-06-04-zabbix/Zabbix_Architecture.png)

아키텍처 초안에서는 시스템 안정성을 위해 프론트엔드, 백엔드, 데이터베이스는 모두 이중화로 구성, 그리고 각 2개의 가용영역에 용도별로 서브넷을 사용하도록 기획했다.

### 아키텍처 최종안
![final_draft](/assets/img/posts/2025-06-04-zabbix/Zabbix_Architecture_final.svg)

최종안에서는 큰 변경사항 없이 RDS 프록시를 추가했다.  
RDS 프록시를 추가한 이유는 Amazon Aurora 클러스터로 구성하게 되면 읽기, 쓰기 인스턴스의 엔드포인트가 개별 생성되기 때문에 재부팅, 장애 등이 발생했을 때 읽기 인스턴스와 쓰기 인스턴스가 서로 전환되기 때문에 문제가 발생할 수 있다.  
RDS 프록시는 쓰기 인스턴스 엔드포인트로만 연결을 해주기 때문에 읽기, 쓰기 인스턴스가 전환되어도 문제가 발생하지 않는다.  

> RDS 프록시를 사용하는 경우 클러스터의 포트는 반드시 데이터베이스 엔진의 기본 포트를 사용해야한다.  
> MySQL: 3306 / PostgreSQL: 5432
{: .prompt-info }

---

## 구성

구성에 사용된 운영체제는 `Ubuntu 24.04`로 구성했다.

### 공통 설정
#### 패키지 설치

네트워크 트러블 슈팅을 위한 도구 설치
```shell
apt install net-tools
```
> 정리를 하면서 찾아보니 net-tools는 2021년 이후로 업데이트가 중단되었다고 한다. 대신 명령어는 다르지만 동일한 기능을 iproute2를 사용하여 대체할 수 있다.  
> iproute2의 경우 기본 설치되어있다.  
{: .prompt-info }

자빅스 패키지 저장소 추가
```shell
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-1+ubuntu24.04_all.deb
dpkg -i zabbix-release_7.0-1+ubuntu24.04_all.deb
apt update
```

호스트네임 변경
```shell
hostnamectl set-hostname <Server Name>
```
### 데이터베이스 

초기 테이블 구성을 위해 데이터베이스로 접근이 가능한 인스턴스에서 실행하면 된다.
```shell
apt install zabbix-sql-scripts -y

zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -h <host> -uzabbix -p zabbix
```

### 백엔드
`zabbix server`, `zabbix agent` 설치
```shell
sudo apt install zabbix-server-mysql zabbix-agent -y
```

자빅스 백엔드 구성정보 수정
```shell
vi /etc/zabbix/zabbix_server.conf
```

```
...
DBHost=<데이터베이스 도메인>	
DBName=zabbix
DBUser=<데이터베이스 사용자 아이디>
DBPassword=<데이터베이스 사용자 패스워드>
DBPort=3306
...
StatsAllowedIP=<자빅스 백엔드 도메인>
...
```
{: file='zabbix_server.conf'}

서비스 재시작
```shell
sudo systemctl restart zabbix-server
```

#### 이중화에서 단일화로 변경 
> 자빅스의 경우 `액티브-스탠바이` 구조로 페일오버 중심의 HA만 지원한다.  
> HA를 설정하게 된다면 운영 오버헤드가 늘어나며 실제로 장애가 발생했을 땐 데이터베이스 성능 문제로 확인과 프록시 서버에서 백엔드와 통신이 불가능하더라도 데이터는 데이터베이스에 저장되었다가 전송되기 때문에 실제 구성에서는 이중화 구성은 제외하기로 결정했다.
{: .prompt-danger }

### 프론트엔드
`zabbix frontend`, `zabbix-nginx`, `zabbix agent` 설치
```shell
apt install zabbix-frontend-php zabbix-nginx-conf zabbix-agent -y
```

자빅스 프론트엔드 구성정보 수정
```shell
vi /usr/share/zabbix/conf/zabbix.conf.php.example
```

```
...
$DB['TYPE']                     = 'MYSQL';
$DB['SERVER']                   = '<데이터베이스 도메인>';
$DB['PORT']                     = '3306';
$DB['DATABASE']                 = 'zabbix';
$DB['USER']                     = '<데이터베이스 사용자 아이디>';
$DB['PASSWORD']                 = '<데이터베이스 사용자 패스워드>';
...
$ZBX_SERVER                  = '<자빅스 백엔드 도메인>';
$ZBX_SERVER_PORT             = '10051';
...
?> 
```
{: file='zabbix.conf.php.example'}

저장 후 파일 복사 및 붙여넣기
```shell
cp /usr/share/zabbix/conf/zabbix.conf.php.example /usr/share/zabbix/conf/zabbix.conf.php
```

Nginx 구성정보 수정
```shell
vi /etc/nginx/conf.d/zabbix.conf
```

```
server {
        server_name     <자빅스 프론트엔드 도메인>;
...
}
```
{: file='zabbix.conf' }

서비스 재시작 및 상태 확인
```shell
systemctl restart nginx php8.3-fpm.service
systemctl status nginx php8.3-fpm.service
```

### 프록시 
`zabbix proxy`, `mysql-server` 설치
```shell
sudo apt install zabbix-proxy-mysql zabbix-sql-scripts mysql-server -y
```
>자빅스에서 제공하는 `zabbix-proxy-mysql`패키지는 문제가 있어 `mysql-server`로 대체했다.
{: .prompt-warning }


프록시 데이터베이스 초기 설정
```shell
sudo mysql -uroot -p
# Mysql 접근 후 쿼리 입력
create database zabbix_proxy character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by '<사용할 데이터베이스 패스워드>';
grant all privileges on zabbix_proxy.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
```

Mysql root 계정 패스워드 설정
```shell
alter user 'root'@'localhost' identified with mysql_native_password by '<사용할 데이터베이스 패스워드>';
flush privileges;
```

패스워드 변경 확인
```shell
sudo mysql -uroot -p
sudo mysql -uzabbix -p
```
각 Mysql 사용자로 로그인이 되는지 확인

데이터베이스 테이블 초기 구성
```shell
cat /usr/share/zabbix-sql-scripts/mysql/proxy.sql | sudo mysql --default-character-set=utf8mb4 -uzabbix -p zabbix_proxy
```

완료 후 Mysql로 접속하여 테이블 생성 확인
```shell
sudo mysql -uzabbix -p
USE zabbix_proxy;
SELECT database();
SHOW tables;
```
아래와 같이 203 행이 있는 것을 확인하면 된다.  

![db_check](/assets/img/posts/2025-06-04-zabbix/db_row.png)


자빅스 프록시 구성정보 설정
```shell
vi /etc/zabbix/zabbix_proxy.conf
```

```
...
Hostname=zbx-zabbix-proxy-01
...
Server=zbx-api.bsgon.com
...
DBPassword=<DBPassword>
```
{: file='zabbix_proxy.conf' }

서비스 재시작
```shell
systemctl restart zabbix-proxy
```

#### 장애 발생
> 프록시 서버에서 로컬에 설치한 MySQL 서버가 원인 모를 이유로 작동이 멈춰 AWS RDS로 재구성했다.
{: .prompt-danger } 

### 자빅스 에이전트 
자빅스는 각 호스트에 에이전트를 설치하여 자빅스 백엔드로 데이터를 보내기 때문에 모니터링이 필요한 서버에는 설치가 필요하다.  

자빅스 에이전트 설치
```shell
apt install zabbix-agent2 zabbix-agent2-plugin-* -y
```

자빅스 에이전트 구성정보 수정
```shell
vi /etc/zabbix/zabbix_agent2.conf
```

```
...
Server=<자빅스 프록시 도메인>
...
ServerActive=<자빅스 프록시 도메인>:10051
...
Hostname=<Hostname>
```
{: file='zabbix_agent2.conf'}

서비스 재시작
```shell
systemctl status zabbix-agent2.service
```

### 그라파나 구성
자빅스 대시보드를 통해 시각화를 할 수 있지만 그래프 스타일과 편의성으로 인해 그라파나를 별도 설치했다.    

그라파나 설치
```shell
wget https://dl.grafana.com/oss/release/grafana_11.0.0_amd64.deb
sudo dpkg -i grafana_11.0.0_amd64.deb
apt update
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
apt update
apt install grafana
```

---
## 회고
2024년 8월에 구축 완료를 했다.  
아키텍처 설계 혼자서 했지만 구축 및 테스트는 팀원들의 도움을 받아 진행했다.


#### 좋았던 점  
팀 내부 프로젝트이지만 자빅스 서버를 신규 구축하면서 대부분 운영 및 유지보수 관련된 업무만 하다가 신규 구축 업무를 진행해보니 매너리즘에 빠지기 전인 나에게는 리프레쉬되는 시간이였다.  


#### 배운 점  
생각보다 자빅스는 많은 기능들을 제공하고 여러 방법으로 데이터를 가져올 수 있다.  
에이전트를 설치할 수 없는 AWS 관리형 서비스의 경우 자빅스에서 제공하는 AWS 템플릿[^1]이 있었지만 자빅스 버전 및 한정된 서비스(AWS RDS, AWS S3, AWS EC2) 등만 제공하기 때문에 일부 모니터링은 CloudWatch 경보를 통해 진행했었다.  
경보 채널은 이메일, 슬랙으로 동일하지만 데이터 수집 및 경보 발생은 중앙화가 아니라 분산되어 있기 때문에 결국에는 오버헤드가 발생하게 된다.  
문제를 해결하기 위해 자빅스에서 공식으로 제공해주는 템플릿을 확인해보니 AWS 자격 증명을 통해 HTTP로 요청하여 CloudWatch 지표 값을 가져온다는 것을 알게 되었다.  
AWS Site to Site VPN는 CloudWatch 경보가 아닌 자빅스에서 데이터 수집, 경보처리를 했고 AWS STS에서 AWS 계정 ID 값을 다른 AWS 관리형 서비스에 맵핑할 수 있도록 구현했다.  

이로인해, 완전한 중앙관리형 모니터링 시스템을 구현할 수 있었다.  

#### 부족한 점
부족한 점보다는 아쉬운 점이 있었다.  
첫번쨰는 백엔드 이중화 설정, 두번째는 컨테이너로 구성이다.

백엔드 이중화 설정은 내가 관련 내용들을 찾아보고 정리하여 진행하려고 했으나 시간 문제로 인해 구성에서 제외하게 되었다.  
본문에서 말한 것과 같이 데이터베이스 성능 문제, 프록시 구성으로 인해 데이터가 수집에는 문제가 없겠지만 그래도 이중화는 반드시 필요하다고 생각했고 구성하지 못해 아쉬웠다.  

자빅스는 컨테이너 이미지로도 제공되기 때문에 Aamzon ECS 또는 Amazon EKS로 구성할 수 있었다.  
그 중 Amazon EKS에 자빅스를 배포하고 싶었지만 팀에서는 쿠버네티스를 다룰 수 있는 사람이 적었기 때문에 EC2 인스턴스에 각 서비스를 배포했다.  
Amazon EKS를 사용했다면 장애 복구, 배포 및 롤백으로 부터 더 자유로웠을 것이다.  
물론 러닝 커브로 인해 도입하지 못했지만 추 후 운영을 진행하면서 Amazon EKS에 자빅스를 배포하는 것이 다음 목표가 되었으면 좋겠다.    


---
## Ref.  
[^1]: [자빅스 공식 AWS 템플릿](https://www.zabbix.com/integration_search?search=aws)



