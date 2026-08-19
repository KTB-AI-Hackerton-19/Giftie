# Giftie BE

[전체 기획서로 돌아가기](README.md)

## 역할

BE는 Giftie의 중심 업무 서버입니다. FE 인증과 모든 업무 API를 담당하고, MySQL에
사용자·사람·선물·캘린더·알림·추천 데이터를 관리합니다. S3 업로드 URL을 발급하며,
AI-Service를 내부 클라이언트로 호출해 결과를 저장 가능한 형태로 변환합니다.

## 현재 기술 스택

- Java 25
- Spring Boot 4.0.6
- Spring MVC
- Spring Security + JWT
- Spring Data JPA
- H2(local), MySQL(prod)
- AWS SDK S3
- SSE
- Springdoc OpenAPI

## 구현된 도메인

| 도메인 | 책임 |
|---|---|
| `User` | 사용자 계정과 인증 |
| `Person` | 선물을 주고받은 사람, 관계, 메모 |
| `GiftRecord` | 받은 선물·금액·날짜·답례 상태 |
| `Category` | 선물 분류와 화면 표시 정보 |
| `ReminderTask` | 알림 예약·발송 상태 |
| `RecommendedGift` | AI 추천 상품 |
| `RecommendationTag` | 추천 태그 |

## 구현된 API

| 영역 | 주요 API |
|---|---|
| 인증 | `POST /api/auth/signup`, `/login`, `/logout`, `/refresh` |
| 대시보드 | `GET /api/dashboard` |
| 선물 기록 | `GET/POST/PATCH/DELETE /api/gift-records`, `POST /extract`, `PATCH /{id}/thanked` |
| 이미지 | `POST /api/gift-assets/presigned-url` |
| 사람 | `GET/POST/PATCH/DELETE /api/people`, `GET /{id}/gift-records` |
| 캘린더 | `GET /api/calendar`, `GET /api/calendar/days/{date}` |
| 알림 | `GET /api/reminders`, `/stream`, `/undelivered`, dispatch API |
| 추천 | `GET /api/recommendations` |
| 검색 | `GET /api/search` |
| 카테고리 | `GET/POST/PATCH/DELETE /api/categories` |
| 상태 | `GET /health` |

Swagger 기본 경로는 `/swagger-ui.html`입니다.

## 주요 서비스

- `AuthService`: 회원가입, 로그인, 토큰 갱신, 로그아웃
- `GiftRecordService`: 선물 생성·추출·수정·조회·답례 완료
- `PersonService`: 사람 관리와 이름 기반 생성/연결
- `CalendarService`: 월·일 단위 일정 조합
- `ReminderTaskService`: 알림 조회, 발송 처리, SSE push
- `RecommendationService`: 사람별 추천 조회와 AI 갱신
- `DashboardService`: 최근 기록, 통계, 에이전트 인사이트 조합
- `S3PresignService`: S3 PUT/GET presigned URL
- `SearchService`: 사람과 기록 통합 검색

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

### 현재 불일치

현재 소스의 `AiExtractionClient`는 `{AI_SERVICE_URL}/extract`,
`AiRecommendationClient`는 `{AI_SERVICE_URL}/recommendations`를 호출합니다.
요청·응답 DTO도 AI-Service의 현재 통합 응답과 다릅니다. 따라서 아직 실제로 연결된
상태가 아니며 실패 시 더미 데이터로 fallback합니다.

### 권장 수정

1. 두 구형 클라이언트를 하나의 `GiftAgentClient`로 통합합니다.
2. 직접 입력은 `/api/v1/agent/from-gift-data`를 호출합니다.
3. 이미지 입력은 `/api/v1/agent/from-image`를 호출합니다.
4. AI 응답의 `gift_data`, `calendar_info`, `noti_info`, `recommend_gift_info` DTO를 만듭니다.
5. `READY`인 데이터만 트랜잭션 정책에 따라 저장합니다.
6. 일부 작업이 `ERROR`여도 성공한 기록은 사용자에게 보여 줄지 정책을 정합니다.
7. 통합 계약 테스트로 필드명과 null 처리 방식을 고정합니다.

## 로컬·운영 설정

```env
AI_SERVICE_URL=http://127.0.0.1:8999
AI_SERVICE_API_KEY=local-development-key
AI_SERVICE_TIMEOUT_MS=90000
```

운영 환경에서는 DB, JWT, AWS 환경변수도 별도로 설정해야 합니다.

## 실행

```bash
cd BE
./gradlew bootRun
```

기본 포트는 `8080`, 기본 프로필은 `local`, 로컬 DB는 H2 메모리 DB입니다.

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

- [ ] `GiftAgentClient`와 통합 DTO 구현
- [ ] 두 AI agent API 계약 테스트
- [ ] AI 성공 결과의 DB 저장 정책 확정
- [ ] 이미지 업로드 → 분석 → 확인 → 저장 흐름
- [ ] 운영 CORS와 JWT secret 교체
- [ ] MySQL migration 전략 적용
- [ ] SSE 운영 프록시 timeout 설정
- [ ] Swagger와 FE 타입 동기화

