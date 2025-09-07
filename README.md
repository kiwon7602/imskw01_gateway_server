mskw01 (Nginx) 게이트웨이 서버 구축

이 문서는 `imskw01` 라즈베리파이(`192.168.0.10`)에 Nginx 리버스 프록시를 설정하는 과정을 안내합니다.

## 1. 사전 준비

- Ubuntu Server가 설치된 라즈베리파이
- 고정 IP 할당
- Docker, Docker Compose 설치

## 2. Nginx 설정

Nginx를 Docker 컨테이너로 실행하여 관리의 용이성을 높입니다.

### 2.1. 작업 디렉토리 생성

```
# 홈 디렉토리 아래에 nginx 작업을 위한 폴더를 생성합니다.
mkdir -p ~/nginx/conf
cd ~/nginx

```

### 2.2. Nginx 설정 파일 작성

`~/nginx/conf/nginx.conf` 파일을 생성하고 아래 내용을 작성합니다.

```
# ~/nginx/conf/nginx.conf

events {}

http {
    server {
        listen 80; # 80번 포트로 들어오는 모든 요청을 수신합니다.

        location / {
            # 모든 요청을 WAS 서버로 전달합니다.
            # imskw02의 IP 주소와 Spring Boot 포트(8080)를 입력합니다.
            proxy_pass http://192.168.0.11:8080;

            # 클라이언트의 실제 IP와 프로토콜 정보를 WAS에 전달하기 위한 헤더 설정입니다.
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}

```

- **해설**:
    - `listen 80;`: 80번 포트(기본 HTTP 포트)로 들어오는 요청을 받습니다.
    - `proxy_pass http://192.168.0.11:8080;`: 받은 모든 요청을 WAS 서버인 `imskw02`의 8080 포트로 그대로 전달(proxy)합니다. 이 부분이 리버스 프록시의 핵심입니다.

### 2.3. Docker Compose 파일 작성

Nginx 컨테이너를 실행하기 위한 `docker-compose.yml` 파일을 `~/nginx` 디렉토리에 생성합니다.

```
# ~/nginx/docker-compose.yml

version: '3'
services:
  nginx:
    image: nginx:latest
    container_name: nginx_reverse_proxy
    ports:
      - "80:80" # 호스트의 80번 포트와 컨테이너의 80번 포트를 연결합니다.
    volumes:
      # 위에서 작성한 nginx.conf 파일을 컨테이너 내부의 설정 파일 경로에 마운트합니다.
      - ./conf/nginx.conf:/etc/nginx/nginx.conf
    restart: always

```

- **해설**:
    - `image: nginx:latest`: 최신 버전의 Nginx 이미지를 사용합니다.
    - `ports: ["80:80"]`: 외부(라즈베리파이)의 80번 포트로 들어온 요청을 컨테이너 내부의 80번 포트로 연결합니다.
    - `volumes`: 호스트의 설정 파일(`~/nginx/conf/nginx.conf`)을 컨테이너의 설정 파일 위치로 연결(마운트)하여, 호스트에서 설정을 변경하면 컨테이너에 즉시 반영되도록 합니다.

## 3. Nginx 실행

`~/nginx` 디렉토리에서 아래 명령어를 실행하여 Nginx 컨테이너를 시작합니다.

```
# -d 옵션은 백그라운드에서 실행하라는 의미입니다.
sudo docker-compose up -d

```

이제 `imskw01` 서버 설정이 완료되었습니다. 외부에서 `http://192.168.0.10`으로 접속하면 모든 요청이 `imskw02` 서버로 전달됩니다.
