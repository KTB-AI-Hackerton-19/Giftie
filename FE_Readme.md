# Giftie FE

[전체 기획서로 돌아가기](README.md)

## 역할

FE는 Giftie의 모바일 중심 사용자 경험을 담당합니다. 사용자는 FE에서 받은 마음을
입력하거나 캡처 이미지를 업로드하고 사람별 선물 이력·캘린더·알림·AI 추천 결과를
확인합니다. 브라우저는 AI-Service를 직접 호출하지 않고 Spring Boot BE만 호출합니다.

## 현재 구현 상태

`../FE`는 Next.js App Router 기반 프로젝트이며 로그인부터 기록·사람·캘린더·마이페이지까지
주요 화면이 실제로 구현되어 BE와 연동된 상태입니다(디자인 원본은 저장소 루트의 Vite
프로토타입을 그대로 따릅니다).

- Next.js 16
- React 19
- TypeScript
- Tailwind CSS 4
- TanStack React Query
- React Hook Form
- Zod

## 구현된 화면

| 라우트 | 내용 |
|---|---|
| `/login` | 로그인 |
| `/` (홈) | 환영 섹션, 통계 카드, AI 인사이트 카드, 최근 기록, 추천 미리보기 |
| `/records` | 카테고리 필터가 있는 전체 기록 목록 |
| `/people` | 사람 목록 |
| `/people/[id]` | 사람별 선물 타임라인 |
| `/calendar` | 월별 캘린더 + 날짜별 목록 |
| `/mypage` | 프로필 조회·수정, 회원탈퇴 |

로그인 화면을 제외한 전체 라우트는 `AuthGuard` + `AppShell`(사이드바/바텀네비/기록
모달/토스트)로 감싸져 있습니다.

## 인증 흐름

로그인 성공 시 access/refresh 토큰을 localStorage에 저장하고 API 요청은 access
토큰을 붙여 보냅니다. 401을 받으면 refresh 토큰으로 자동 재발급을 한 번 시도한
뒤 재요청하고 그마저 실패하면 토큰을 지우고 로그인 화면으로 보냅니다. 여러 탭에서
로그인 상태가 동기화됩니다.

## 이미지 업로드 → AI 추출 흐름

1. 사진 선택 후 "AI로 정리하기" 클릭
2. presigned URL 발급 → S3에 직접 PUT 업로드
3. 발급받은 `imageKey`로 `POST /api/gift-records/extract` 호출 → BE가 AI-Service를
   거쳐 만든 DRAFT 기록을 반환
4. 화면에서 내용을 검토·수정
5. 저장 시 draft가 있으면 `PATCH /api/gift-records/{id}`(확정), 사진 없이 직접
   입력했다면 `POST /api/gift-records`(신규)

## BE 연동 방식

- BE는 `https://api.giftie.site`에 배포되어 있고 CORS가 열려 있어, `NEXT_PUBLIC_API_BASE_URL`을
  이 주소로 지정해 브라우저에서 직접 호출합니다.
- `next.config.ts`의 `API_PROXY_TARGET` rewrite는 과거 BE가 인증 경로 OPTIONS preflight를
  401로 막던 시절의 우회책이었던 legacy 폴백입니다. 지금은 안 쓰지만, CORS가 다시 막히거나
  로컬 BE·ngrok으로 붙어야 할 때를 위해 남겨 뒀습니다(`NEXT_PUBLIC_API_BASE_URL`을 비우고
  `API_PROXY_TARGET`을 채우면 전환됩니다).
- 모든 API 응답은 BE의 `{success, data, error}` 포맷을 공통 클라이언트가 벗겨서
  돌려줍니다.
- React Query로 서버 상태를 관리하고 mutation 성공 시 관련 query를 무효화합니다.
- SSE는 `/api/reminders/stream`을 구독해 실시간 알림을 토스트로 띄웁니다.

## 주요 BE API 사용 영역

| 화면 | API |
|---|---|
| 인증 | `/api/auth/*` |
| 내 프로필 | `/api/users/me` |
| 대시보드 | `GET /api/dashboard` |
| 선물 기록 | `/api/gift-records/*`, `POST /extract` |
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

- [x] 모바일 폭 우선 레이아웃
- [x] 로그인 및 토큰 갱신
- [x] 직접 입력과 이미지 입력
- [x] 사람별 리스트와 상세 이력
- [x] 월 캘린더와 날짜별 목록
- [x] SSE 알림
- [x] 마이페이지(프로필 수정, 회원탈퇴)
- [ ] AI 추천 상품 및 메시지 화면 반영 (BE 추천 연동 완료 후)
- [ ] 로딩·빈 상태·부분 실패 UI 다듬기
- [ ] BE 통합 테스트
