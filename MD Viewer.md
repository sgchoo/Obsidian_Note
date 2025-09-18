### CentOS 배포 가이드: Docker/Compose 설치 → 이미지 빌드 → Compose 실행

  

이 문서는 CentOS 환경에서 Docker와 Docker Compose 설치, 애플리케이션 이미지 빌드, `docker-compose.prod.yml`을 통한 실행까지를 설명합니다.

  

- 테스트된 대상: CentOS 7/8/Stream (root 또는 sudo 권한 필요)

- 애플리케이션 포트: 8080

- 외부 네트워크: `star-savior-network` (compose에서 external로 사용)

  

---

  

### 0) 준비

  

- zip파일을 압축 해제한 후, docker-compose.prod.yml, star-savior-server.tar, .env, .env.prod 파일들을 확인한다.

- 운영 서버 인스턴스에 하나의 디렉토리에 넣어준다.

- 옮긴 디렉토리에서 아래 명령어들을 실행한다.

  

---

  

### 1) Docker 설치

  

- 시스템 업데이트 및 필요한 패키지 설치

  

```bash

sudo yum update -y

sudo yum install -y yum-utils device-mapper-persistent-data lvm2

```

  

---

  

- Docker 공식 저장소 추가

  

```bash

sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

```

  

---

  

- Docker 설치

  

```bash

sudo yum install -y docker-ce docker-ce-cli containerd.io

```

  

---

  

- Docker 서비스 시작 및 자동 시작 설정

  

```bash

sudo systemctl start docker

sudo systemctl enable docker

```

  

---

  

- 현재 사용자를 docker 그룹에 추가 (선택사항)

  

```bash

sudo usermod -aG docker $USER

# 이후 로그아웃 후 다시 로그인해야 적용됩니다.

```

  

---

  

- Docker 설치 확인

  

```bash

docker --version

sudo docker run hello-world

```

  

---

  

### 2) Docker Compose 설치

  

방법 1: 바이너리 직접 다운로드 (권장)

  

```bash

# 최신 버전 확인 후 설치 (예: v2.24.0)

sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

```

  

---

  

```bash

# 실행 권한 부여

sudo chmod +x /usr/local/bin/docker-compose

```

  

---

  

```bash

# 심볼릭 링크 생성 (선택사항)

sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

```

  

---

  

- 확인:

  

```bash

docker version

  

```

  

---

  

### 3) 네트워크 셋팅

  

```bash

docker network create star-savior-network

```

  

---

  

### 4) Docker 이미지 로드

  

- Docker 이미지 빌드:

  

```bash

docker load -i star-savior-server.tar

```

  

---

  

### 5) docker-compose.prod.yml 실행

  

- 실행:

  

```bash

docker compose -f docker-compose.prod.yml up -d

# 또는(플러그인 미사용)

# docker-compose -f docker-compose.prod.yml up -d

```

  

- 로그 확인:

  

```bash

docker logs -f star-savior-server-stage

```

  

- 종료/정리:

  

```bash

# 서버 애플리케이션을 종료하고 싶다면

docker compose -f docker-compose.stage.yml down

```

  

---

  

### 6) Prisma 마이그레이션/시드

  

- 컨테이너 내부 진입:

  

```bash

docker exec -it star-savior-server /bin/sh"

```

  

- Prisma 마이그레이션 및 적용

  

```bash

npx prisma migrate deploy --schema=./prisma

npx prisma generate --schema=./prisma

```

  

- 시드 데이터 저장

  

```bash

# 임시 관리자 계정과 부서 데이터를 저장합니다.

pnpm run prisma:seed

```

  

---

  

### 7) Nginx 셋팅

  

- Nest 백엔드 애플리케이션 conf 파일 생성

  

```bash

location /api/ {

proxy_pass http://127.0.0.1:8080;

proxy_set_header Host $host;

proxy_set_header X-Real-IP $remote_addr;

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

proxy_set_header X-Forwarded-Proto $scheme;

  

if ($host ~* "^(dev\.starsavior\.com|admin-dev\.starsavior\.com)$") {

add_header Access-Control-Allow-Origin "https://$host";

add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";

add_header Access-Control-Allow-Headers "Content-Type, Authorization";

}

  
  

if ($request_method = "OPTIONS") {

add_header Access-Control-Allow-Origin "https://$host";

add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";

add_header Access-Control-Allow-Headers "Content-Type, Authorization";

add_header Content-Length 0;

add_header Content-Type text/plain;

return 200;

}

}

```

  

---

  

- 기존 Nginx 설정 파일에 지시어 삽입

  

```bash

location / {

...

  

include proxy.conf;

}

```

  

---