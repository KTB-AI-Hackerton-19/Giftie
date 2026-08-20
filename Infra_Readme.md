# Giftie Infrastructure

선물·부조금 기록과 답례 시점을 AI가 관리해주는 서비스 **Giftie**의 인프라 구성입니다.

## Architecture

```
User → Vercel (FE)
         │  API Calls (HTTPS)
         ▼
   AWS Cloud
   ├── EC2 - 백엔드 호스트
   │     Nginx (Docker, TLS 종료) ← certbot (Let's Encrypt)
   │     Nginx --HTTPS--> Backend (Spring API Server, Docker)
   │     Backend ↔ MySQL (Docker)
   │     Backend → S3 (이미지 저장, presigned URL)
   │     Backend → EC2 - AI Worker
   │     Backend → External APIs (Google Calendar API / Notifications)
   │
   └── EC2 - AI Worker
         python (Bedrock API Key)
         python → AWS Bedrock (Claude Haiku)
```

<img width="1024" height="692" alt="image" src="https://github.com/user-attachments/assets/350b9852-95e8-47f0-9cab-cc6de4356909" />


FE와 BE는 서로 다른 서브도메인으로 분리 배포되며, 각각 독립적으로 HTTPS를 종료합니다.

## Tech Stack

| 영역 | 기술 |
| --- | --- |
| FE 배포 | Vercel |
| BE 배포 | AWS EC2, Docker Compose |
| Reverse Proxy / TLS | Nginx, Let's Encrypt (certbot) |
| DB | MySQL |
| Object Storage | AWS S3 (presigned URL 업로드) |
| Container Registry | GitHub Container Registry (GHCR) |
| CI/CD | GitHub Actions |
| AI | AWS Bedrock (Claude Haiku) |
| 외부 연동 | Google Calendar API |

## Components

### FE — Vercel

Next.js 프로젝트를 Vercel에 배포합니다. `main` 브랜치에 push되면 자동으로 빌드·배포되며, 커스텀 도메인을 통해 서비스됩니다.

### BE — EC2

단일 EC2 인스턴스 위에서 Docker Compose로 아래 컨테이너를 함께 운영합니다.

| 서비스 | 역할 |
| --- | --- |
| `nginx` | 리버스 프록시, TLS 종료 |
| `certbot` | 인증서 발급 및 자동 갱신 |
| `backend` | Spring Boot API 서버 |
| `mysql` | 데이터 저장소 |

### Object Storage — S3

이미지 업로드는 presigned URL 방식을 사용합니다. BE가 업로드 권한을 서명해 발급하면, FE가 그 URL로 S3에 직접 업로드해 BE 서버가 파일 트래픽을 직접 처리하지 않도록 구성했습니다.

### AI Worker — EC2

별도 EC2 인스턴스에서 Python 워커가 AWS Bedrock(Claude Haiku)을 호출해 답례 판단 및 선물 추천 로직을 처리합니다.

### External APIs

Google Calendar API를 통해 답례 준비 일정을 사용자 캘린더에 등록합니다.

## Deployment

GitHub Actions를 통해 아래 파이프라인으로 BE를 배포합니다.

```
main push → GitHub Actions
   → Docker 이미지 빌드
   → GHCR(GitHub Container Registry)에 push
   → EC2에 SSH 접속, 최신 이미지 pull 후 컨테이너 재기동
```

FE는 Vercel의 Git 연동을 통해 별도 파이프라인으로 자동 배포됩니다.


GitHub Actions를 통해 아래 파이프라인으로 AI-Service를 배포합니다.

```
main push → GitHub Actions
   → GHCR(GitHub Container Registry)에 push
   → EC2에 SSH 접속, 최신 Git Repo pull 후 restart.sh 파일 가동
```

## Directory Structure (EC2)

```
~/giftie/
├── docker-compose.yml
├── .env
├── nginx/
│   └── conf.d/
│       └── api.conf
└── certbot/
    ├── conf/
    └── www/
```

## Environment Variables

| 변수 | 설명 |
| --- | --- |
| `DB_NAME` / `DB_USERNAME` / `DB_PASSWORD` | MySQL 접속 정보 |
| `MYSQL_ROOT_PASSWORD` | MySQL 컨테이너 초기화용 |
| `AWS_S3_BUCKET_NAME` / `AWS_REGION` / `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | S3 presigned URL 발급 |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` / `GOOGLE_REDIRECT_URI` | Google OAuth2 / Calendar 연동 |
| `JAVA_OPTS` | JVM 힙 설정 |
