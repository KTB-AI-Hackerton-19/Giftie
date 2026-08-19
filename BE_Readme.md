# Giftie BE

[전체 기획서로 돌아가기](README.md)

## 역할

BE는 Giftie의 중심 업무 서버입니다. FE 인증과 모든 업무 API를 담당하고 DB에
사용자·사람·선물·캘린더·알림·추천 데이터를 관리합니다. S3 업로드 URL을 발급하며
AI-Service를 내부 클라이언트로 호출해 결과를 저장 가능한 형태로 변환합니다.

## 현재 기술 스택

- Java 25
- Spring Boot 4.0.6
- Spring MVC
- Spring Security + JWT (Access/Refresh)
- Spring Data JPA
- H2(local), MySQL(prod)
- AWS SDK S3
- SSE
- Springdoc OpenAPI

## 구현된 도메인

| 도메인 | 책임 |
|---|---|
| `User` | 사용자 계정, 인증, 표시 이름·프로필 이미지 |
| `Person` | 선물을 주고받은 사람, 관계, 메모, 생일 |
| `GiftRecord` | 받은 선물·금액·날짜·답례 상태 (DRAFT/CONFIRMED) |
| `Category` | 선물 분류와 화면 표시 정보 (User별 소유, DB 기반이라 추가 시 재배포 불필요) |
| `ReminderTask` | 알림 예약·발송·표시 상태 |
| `RecommendedGift` | AI 추천 상품 캐시 |

## 구현된 API

| 영역 | 주요 API |
|---|---|
| 인증 | `POST /api/auth/signup`, `/login`, `/logout`, `/refresh` |
| 내 프로필 | `GET/PATCH/DELETE /api/users/me` (프로필 이미지, 회원탈퇴 포함) |
| 대시보드 | `GET /api/dashboard` |
| 선물 기록 | `GET/POST/PATCH/DELETE /api/gift-records`, `POST /extract`, `PATCH /{id}/thanked` |
| 이미지 | `POST /api/gift-assets/presigned-url` |
| 사람 | `GET/POST/PATCH/DELETE /api/people`, `GET /{id}/gift-records` |
| 캘린더 | `GET /api/calendar`, `GET /api/calendar/days/{date}` |
| 알림 | `GET /api/reminders`, `/stream`(SSE), `/undelivered`, `POST /{id}/delivered`, dispatch API |
| 추천 | `GET /api/recommendations` |
| 검색 | `GET /api/search` |
| 카테고리 | `GET/POST/PATCH/DELETE /api/categories` |
| 상태 | `GET /health` |

Swagger 기본 경로는 `/swagger-ui.html`이며 응답은 전부 `ApiResponse<T>`
(성공: `data`만, 실패: `error`만) 공통 포맷을 씁니다.

## 알림 발송 (SSE)

10분 주기가 아니라 **오전 9시~밤 9시 매시 정각**(`0 0 9-21 * * *`) 스케줄러가 예정일
도래한 알림을 `SENT`로 바꾸며 접속 중인 사용자에게 SSE로 즉시 push합니다. `PENDING`인
것만 집어가는 멱등 처리라 여러 번 돌아도 중복 발송이 없습니다. `EventSource`가
`Authorization` 헤더를 못 붙이는 제약 때문에 `/stream`은 `?token=<accessToken>` 쿼리로
인증을 허용합니다. 접속하지 않은 동안 발송된 알림은 다음 접속 시 SSE 연결 직후
흘려보내 유실을 막습니다.

## AI-Service 통합

### AI-Service 실제 계약

```http
POST {AI_SERVICE_URL}/api/v1/agent/from-gift-data
X-API-KEY: {AI_SERVICE_API_KEY}
Content-Type: application/json
```

```json
{
  "gift_data": {
    "gift_name": "스타벅스 케이크",
    "gift_price": 35000,
    "age": 29,
    "person_name": "김민수",
    "relationship": "친구",
    "received_at": "2026-08-19",
    "target_date": null
  }
}
```

이미지는 다음 API로 전달합니다.

```http
POST {AI_SERVICE_URL}/api/v1/agent/from-image
```

```json
{
  "image_url": "https://presigned-s3-url"
}
```

### 현재 연동 상태

**이미지 추출(`AiExtractionClient`)은 이미 올바르게 연결되어 있습니다** —
`{AI_SERVICE_URL}/api/v1/agent/from-image`를 호출하고 응답의 `gift_data.payload`를
읽어 저장 가능한 형태로 변환합니다. `AI_SERVICE_URL` 미설정이나 호출 실패 시에만
하드코딩 더미로 폴백합니다.

**추천(`AiRecommendationClient`)은 아직 옛 계약을 씁니다.** `{AI_SERVICE_URL}/recommendations`를
호출하는데, AI-Service의 실제 API는 `/api/v1/agent/recommend`이고 요청·응답 DTO도
다릅니다. 따라서 추천은 현재 항상 하드코딩 더미 3건으로 동작합니다.

### 권장 수정

1. `AiRecommendationClient`가 `POST /api/v1/agent/recommend`를 호출하도록 경로와
   요청 DTO(`age`, `gender`, `budget_min/max`, `categories` 등)를 AI-Service 계약에 맞춥니다.
2. 응답의 `recommend_gift_info`(`status`가 `SUCCESS`/`ERROR`/`SKIPPED`)를 처리하는
   DTO를 새로 만듭니다. `SKIPPED`는 오류가 아니라 "이 기록에는 답례 선물 추천이
   맞지 않음"이므로 화면에 실패로 보여주면 안 됩니다.
3. 통합 계약 테스트로 필드명과 null 처리 방식을 고정합니다.

## 로컬·운영 설정

```env
AI_SERVICE_URL=http://127.0.0.1:8999
AI_SERVICE_API_KEY=local-development-key
AI_SERVICE_TIMEOUT_MS=90000
```

로컬 `application.yml`의 기본값은 현재 AI-Service를 ngrok 터널 주소로 가리킵니다
(개발 중 임시 배선이므로 ngrok 세션이 바뀌면 값도 함께 갱신해야 합니다).
운영 환경에서는 DB, JWT, AWS 환경변수도 별도로 설정해야 합니다.

## 실행

```bash
cd BE
./gradlew bootRun
```

기본 포트는 `8080`, 기본 프로필은 `local`, 로컬 DB는 H2입니다.

## AWS 배포 시 책임

- ALB 뒤에서 HTTPS API 제공
- private subnet의 RDS MySQL 연결
- S3 IAM 권한과 private bucket 구성
- AI-Service private 주소 설정
- Secrets Manager/Parameter Store에서 비밀값 주입
- FE 도메인만 허용하도록 CORS 수정
- JWT, API key, presigned URL 로그 마스킹
- AI 최초 모델 적재를 고려한 충분한 read timeout 설정

## BE 완료 조건

- [x] 인증(회원가입/로그인/로그아웃/토큰 갱신), 사람·선물기록·카테고리 CRUD
- [x] 캘린더, 대시보드, 통합검색
- [x] 알림 스케줄러 + SSE 실시간 push
- [x] 내 프로필 조회·수정·회원탈퇴
- [x] presigned URL 발급, 이미지 추출 AI 연동
- [ ] `AiRecommendationClient`를 AI-Service의 `/api/v1/agent/recommend` 계약에 맞게 수정
- [ ] AI 성공 결과의 DB 저장 정책 확정 (부분 실패 시 무엇을 저장할지)
- [ ] 운영 CORS와 JWT secret 교체
- [ ] MySQL migration 전략 적용
- [ ] SSE 운영 프록시 timeout 설정
- [ ] Swagger와 FE 타입 동기화
