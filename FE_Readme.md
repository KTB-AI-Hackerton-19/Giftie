# Giftie FE

[전체 기획서로 돌아가기](README.md)

## 역할

FE는 Giftie의 모바일 중심 사용자 경험을 담당합니다. 사용자는 FE에서 받은 마음을
입력하거나 캡처 이미지를 업로드하고, 사람별 선물 이력·캘린더·알림·AI 추천 결과를
확인합니다. 브라우저는 AI-Service를 직접 호출하지 않고 Spring Boot BE만 호출합니다.

## 현재 구현 상태

`../FE`는 Next.js App Router 기반 프로젝트이며 현재 기본 시작 화면 단계입니다.

- Next.js 16.3.1
- React 19.2.8
- TypeScript
- Tailwind CSS 4
- TanStack React Query
- React Hook Form
- Zod

`src/app/providers.tsx`에 Query Client 공급자 골격이 있고, 실제 Giftie 페이지·컴포넌트·API
호출 코드는 앞으로 구현해야 합니다.

## 구현할 주요 화면

### 홈·대시보드

- 최근 받은 마음
- 곧 필요한 답례와 알림
- AI가 제안하는 다음 행동
- 새 마음 기록 버튼

### 마음 기록

- 직접 입력: 선물명, 가격, 나이, 사람, 관계, 받은 날짜, 목표 날짜
- 이미지 입력: 캡처 업로드 → AI 추출 → 사용자 확인·수정
- 빈 선택값은 `null` 또는 `""`로 전송 가능

### 사람 목록과 상세

- 어떤 사람이 선물했는지 목록 표시
- 사람을 선택하면 해당 인물의 선물 이력을 최신순으로 표시
- 관계·메모·생일 수정

### 캘린더

- 최초 진입 시 오늘이 속한 월 표시
- 선물·답례가 있는 날짜 강조
- 날짜 선택 시 관련 사람과 선물 목록 표시
- 이전/다음 달 이동

### 알림

- 예정 알림 목록
- SSE 구독을 통한 실시간 알림
- 답례 완료 처리

### 추천 결과

- 추천 가격 범위
- 카테고리, 점수, 이유
- 실제 상품명, 가격, 이미지, 구매 링크
- `IN_RANGE`와 `NEAREST` 구분
- AI 감사 메시지 복사

## 권장 프론트 구조

```text
src/
├── app/
│   ├── login/
│   ├── dashboard/
│   ├── gifts/
│   ├── people/
│   ├── calendar/
│   └── recommendations/
├── components/
├── features/
├── lib/
│   ├── api.ts
│   └── auth.ts
├── schemas/
└── types/
```

## BE 연동 원칙

- `NEXT_PUBLIC_API_BASE_URL`에는 Spring Boot 주소만 설정합니다.
- 로그인 후 JWT access token을 API 요청에 첨부합니다.
- React Query로 서버 상태를 관리하고 mutation 성공 시 관련 query를 무효화합니다.
- Zod 스키마는 BE DTO와 동일한 필수/선택 조건으로 맞춥니다.
- 이미지 업로드는 presigned URL 발급 → S3 PUT → 이미지 key 기록 요청 순서로 처리합니다.
- SSE는 `/api/reminders/stream`을 구독합니다.

## 주요 BE API 사용 영역

| 화면 | API |
|---|---|
| 인증 | `/api/auth/*` |
| 대시보드 | `GET /api/dashboard` |
| 선물 기록 | `/api/gift-records/*` |
| 이미지 업로드 | `POST /api/gift-assets/presigned-url` |
| 사람 | `/api/people/*` |
| 캘린더 | `/api/calendar`, `/api/calendar/days/{date}` |
| 알림 | `/api/reminders/*`, `/api/reminders/stream` |
| 추천 | `GET /api/recommendations` |
| 검색 | `GET /api/search` |
| 카테고리 | `/api/categories/*` |

## 실행

```bash
cd FE
npm install
npm run dev
```

기본 주소는 `http://localhost:3000`입니다.

## FE 완료 조건

- [ ] 모바일 폭 우선 레이아웃
- [ ] 로그인 및 토큰 갱신
- [ ] 직접 입력과 이미지 입력
- [ ] 사람별 리스트와 상세 이력
- [ ] 월 캘린더와 날짜별 목록
- [ ] SSE 알림
- [ ] AI 추천 상품 및 메시지 표시
- [ ] 로딩·빈 상태·부분 실패 UI
- [ ] BE 통합 테스트

