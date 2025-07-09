---
title: 나만의 n8n 서버 만들기
date: 2025-05-29 12:00:00 +0900
categories: [knowledege, article]
tags: [n8n]
---
# 개요
n8n은 워크플로 자동화 및 데이터 통합을 위한 오프 소스 도구이다.  
기존에 Zaiper나 Make를 사용해봤지만 나에게는 사용법이 어려워 사용하지 않았었다.  
요즘 AI 관련 새로운 소식을 보고 싶었지만 국내 자료보다 해외 자료가 빠르고 정확하여 이를 자동화를 통해 자료를 보기 위해 n8n을 구축해보았다.


# 서버 환경
> OS: Amazon Linux 2023 ARM  
> EC2 인스턴스 유형: t4g.medium  
{: .prompt-tip }

AWS에서는 ARM, AMD 아키텍처 모두 지원되지만 ARM의 경우 AMD 아키텍처 대비 비용이 저렴하기 때문에 ARM 아키텍처로 선택했다.  

# n8n 설치
n8n은 SaaS로도 제공되지만 자체 호스팅하여 사용할 수도 있다.  
SaaS 같은 경우 유료로 제공되며 제 3자 서버에 데이터가 저장되기 때문에 자체 호스팅 방법으로 구축을 해보았다.  

n8n의 경우 node.js 기반으로 동작되는 애플리케이션이기 때문에 node.js 설치는 필수이다.  
node를 직접 설치하지 않도 `nvm`

### nvm 설치 및 node.js 설치

```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
nvm install 22
```

### node.js 설치 확인
```shell
node -v
nvm current
```
두개의 버전이 동일한지 확인해준다.  

### n8n 설치
공식문서[^1]를 확인하면 설치방법을 확인할 수 있다.  
```shell
npm install n8n -g
```
명령어를 실행하면 설치가 진행된다.  
설치까지 생각보다 시간이 오래 소요된다.  

설치가 완료됐다면 아래 명령어를 입력해 n8n이 정상 실행되는지 확인하면 된다.

### 설치 확인
```shell
n8n
```

정상적으로 실행이 된다면 localhost:5678로 접근해서 확인할 수 있다.
### 타임존 변경
```shell
sudo timedatectl set-timezone Asia/Seoul
```

# HTTPS 설정
외부에서 안전하게 접근하기 위해서 HTTP보다는 HTTPS를 사용하는 것이 좋다.  
퍼블릭 도메인은 필수로 있어야 한다.   
Amazon EC2에 n8n을 배포하여 ALB를 통해 ACM 인증서를 사용할 수도 있지만 비용을 최소한으로 사용하기 위해 nginx를 사용해서 구축해보았다.  

### Nginx 설치
```shell
yum install nginx
```

### 구성 정보 설정
```shell
vi /etc/nginx/conf.d/n8n.conf
```  
   

```
server {
    server_name n8n.<domain>.com;

    location / {
        proxy_pass http://localhost:5678;
        rewrite ^/n8n/(.*)$ /$1 break;


        # WebSocket Support Header
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # General Proxy Header
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 300s;
        proxy_send_timeout 300s;

    }

}
```
{: file='n8n.conf'}

### certbot, python-certbot-nginx 설치 
```shell
dnf install certbot python3-certbot-nginx -y
```

### 인증서 발급
```shell
certbot --nginx -d <domain>
```

### 자동 갱신 확인
```shell
certbot renew --dry-run
```

등록이 완료된다면  `managed by Certbot` 주석이 달린 내용이 추가된다.
```
server {
    server_name n8n.<domain>.com;

    location / {
        proxy_pass http://localhost:5678;
        rewrite ^/n8n/(.*)$ /$1 break;


        # WebSocket Support Header
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # General Proxy Header
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 300s;
        proxy_send_timeout 300s;

    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/n8n.<domain>.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/n8n.<domain>.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = n8n.<domain>.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    server_name n8n.<domain>.com;
    return 404; # managed by Certbot


}
```
{: file='n8n.conf'}

HTTPS로 접근하는 SSL 인증서가 적용되도록 부분과 HTTP로 접근해도 HTTPS로 301 리다이렉트 시키는 부분이 추가된 것을 확인할 수 있다.

# systemd 등록
n8n은 기본적으로 foreground로 실행이 된다.  
nohup이나 script를 작성하여 서비스를 기동할 수 있지만 운영 관점에서는 오버헤드가 커질 뿐이다.  
systemd에 등록을 한다면 systemctl 명령어를 사용해서 서비스를 중지, 시작, 재시작, 재부팅 후 시작까지 설정할 수 있으므로 보통의 경우에서는 사용하는 것이 좋다.  

### n8n 서비스 등록
```shell
vi /etc/systemd/system/n8n.service
```

```
[Unit]
Description=n8n automation tool
After=network.target

[Service]
ExecStart=/root/.nvm/versions/node/v22.15.0/bin/n8n
Restart=always
User=root
Environment=PATH=/root/.nvm/versions/node/v22.15.0/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
Environment=NODE_ENV=production
Environment=N8N_PORT=5678
Environment=N8N_HOST=0.0.0.0
Environment=N8N_EDITOR_BASE_URL=https://n8n.<domain>.com
Environment=WEBHOOK_URL=https://n8n.<domain>.com
Environment=GENERIC_TIMEZONE=Asia/Seoul
Environment=QUEUE_HEALTH_CHECK_ACTIVE=true
Environment=N8N_METRICS=true
WorkingDirectory=/root

[Install]
WantedBy=multi-user.target
```
{: file='n8n.service'}

저장하고 리로드를 하면 서비스에 등록이 된다.  

### 리로드 및 n8n 서비스 시작
```shell
systemctl daemon-reload  
systemctl start n8n
```

정상적으로 실행되게 된다면 내가 설정한 도메인으로 HTTPS 접근이 가능하며 브라우저의 자물쇠를 클릭하여 인증서 등록이 잘되었는지 확인할 수 있다.

![img.png](/assets/img/posts/2025-05-09-n8n/img.png)

---
## Ref.
[^1]: [n8n 설치 공식 문서](https://docs.n8n.io/hosting/installation/npm/)
