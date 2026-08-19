# Giftie AI Service

[전체 기획서로 돌아가기](README.md)

실제 실행법, 전체 응답 예시, 함수 시그니처와 구현 지점은
`../AI-Service/README.md`에 가장 상세하게 정리되어 있습니다. 이 문서는 그 요약입니다.

## 역할

AI-Service는 FastAPI 오케스트레이터입니다. Spring Boot가 전달한 직접 선물데이터 또는
이미지 URL을 공통 `GiftData`로 만들고 네 독립 작업(기록·캘린더·알림·추천)을 비동기로
실행해 하나의 JSON으로 반환합니다.

## 공개 API

| Method | Path | 역할 |
|---|---|---|
| POST | `/api/v1/agent/from-gift-data` | 구조화된 선물데이터로 네 작업 실행 (준비만, 캘린더 미등록) |
| POST | `/api/v1/agent/from-image` | 이미지 분석 후 네 작업 실행 (준비만, 캘린더 미등록) |
| POST | `/api/v1/agent/confirm` | 사용자 검토·수정본을 확정하고 캘린더에 실제 등록 |
| POST | `/api/v1/agent/recommend` | 추천만 단독 실행 (이미지·캘린더·알림 재실행 없이) |

네 API 모두 `X-API-KEY` 헤더가 필요합니다.

## 처리 구조

```text
직접 GiftData ───────────────────────────┐
                                        ├─ 공통 GiftData
이미지 URL → 이미지 분석(Bedrock/vLLM) ──┘
                   │
                   ├─ 선물 기록 데이터 준비
                   ├─ 캘린더 초안 준비 (등록은 /confirm 에서만)
                   ├─ 알림 데이터 준비
                   └─ 답례 추천·메시지 (선물·청첩장만 실행)
                          └─ Tavily 카테고리별 실상품 검색
                   │
                   └─ 비동기 결과 병합 → JSON
```

네 작업은 `asyncio.gather(..., return_exceptions=True)`로 동시에 실행합니다. 한 작업이
실패해도 나머지 결과는 유지하고 실패한 작업만 `status: "ERROR"`로 반환합니다.

## 실제 구현 상태

Mock으로 남은 부분 없이, 아래 전부 실제로 동작합니다.

| 기능 | 상태 |
|---|---|
| 이미지에서 GiftData 추출 | 실제 실행 (Bedrock Claude Sonnet 또는 vLLM Gemma4-12B-QAT) |
| 답례 추천 카테고리·가격대·메시지 | 실제 실행 (Bedrock Claude Haiku 4.5 또는 vLLM) |
| 실제 상품 검색 | 실제 실행 (Tavily, 판매가까지 검증) |
| 선물 기록·알림 저장용 JSON | 실제 실행 |
| Google Calendar 등록 | 실제 실행 (MCP 서버, `/confirm`에서 사용자 승인 후에만) |

**기본 실행 경로는 Amazon Bedrock**(`MODEL_BACKEND=bedrock`)이며 GPU나 로컬 모델 적재가
필요 없습니다. 자체 GPU가 있으면 vLLM 서버(Gemma4-12B-QAT)로 대체할 수 있습니다.

## 추천이 필요 없는 입력 구분

현금·부조금·영수증에는 답례 "선물"이라는 개념이 어울리지 않습니다. AI-Service는 받은
기록의 종류(또는 사용자가 업로드 화면에서 직접 고른 `category`)를 보고 추천 대상이
아니면 모델을 아예 호출하지 않고 `recommend_gift_info.status = "SKIPPED"`를 돌려줍니다.
청첩장·부고장(`event_invitation`)은 답례품이 아니라 축의금·조의금 적정 수준을 안내하므로
추천 대상에 포함합니다. 프론트는 `SKIPPED`를 오류로 표시하면 안 됩니다.

## 추천 정책

- 받은 가격의 80~120%를 기본 범위로 사용 (금액대별 100/1,000/10,000원 단위 반올림)
- 여러 사람에게 받았다면 각 금액의 최저 80%~최고 120%로 범위를 넓힘
- `age`가 `0`, `"0"`, 빈 문자열 또는 `null`이면 미입력 처리
- 카테고리는 허용 목록 안에서만, 모델 JSON 파싱 실패 시 안전한 fallback 추천으로 대체

## 실상품 검색 정책

- 모델이 카테고리·상품 유형·가격 범위를 정하면 검색은 파이프라인이 결정론적으로 실행 (모델에게 검색 툴을 주지 않음)
- 신뢰할 수 있는 국내 쇼핑몰(쿠팡, 카카오 선물하기, 네이버 쇼핑 등)만 검색
- 검색·목록 페이지가 아닌 개별 상품 상세페이지만 통과
- 검색 스니펫이 아니라 상품 페이지 본문에서 확인한 판매가만 신뢰(`price_verified`)
- 가격 범위 안 상품 우선, 없으면 가장 가까운 가격의 상품 하나를 대안으로 반환
- 검색 실패 시 `products: []`로 응답하며 카테고리 추천 자체는 그대로 나감

## 로컬 실행

```bash
cd AI-Service
source .venv-runtime/bin/activate
uvicorn app.main:app --host 0.0.0.0 --port 8999
```

`./start.sh` / `./stop.sh` / `./restart.sh`로도 띄우고 내릴 수 있습니다(백그라운드 실행,
날짜별 로그, 설치돼 있으면 ngrok 터널까지 함께 관리).

- Swagger: `http://127.0.0.1:8999/docs`
- 테스트: `pytest -q`

## 주요 환경변수

```env
API_KEY=local-development-key
MODEL_BACKEND=bedrock
BEDROCK_API_STYLE=invoke
BEDROCK_REGION=us-east-1
BEDROCK_MODEL_ID=us.anthropic.claude-haiku-4-5-20251001-v1:0
BEDROCK_VISION_MODEL_ID=global.anthropic.claude-sonnet-4-6
AWS_BEARER_TOKEN_BEDROCK=
TAVILY_ENABLED=true
TAVILY_API_KEY=
CALENDAR_MCP_URL=http://localhost:8300/mcp
GOOGLE_ACCESS_TOKEN=
CALENDAR_AUTO_REGISTER=false
```

전체 목록과 각 값의 의미는 `../AI-Service/README.md`의 "환경 설정" 절을 참고합니다.

## GPU 서버로 전환하는 경우

- `MODEL_BACKEND=vllm`로 바꾸고 자체 GPU에 vLLM 서버(Gemma4-12B-QAT)를 띄웁니다.
- AI-Service는 public endpoint 대신 private subnet에 두고 BE에서만 호출합니다.
- BE와 같은 `AI_SERVICE_API_KEY`를 Secrets Manager로 주입합니다.

## BE와의 연동 상태

BE의 `AiExtractionClient`는 이미 `/api/v1/agent/from-image`를 올바르게 호출합니다.
반면 `AiRecommendationClient`는 아직 존재하지 않는 `/recommendations` 경로를 호출하고
있어 실패 시 더미 추천으로 폴백합니다. `/api/v1/agent/recommend`로 바꾸는 작업이
남아 있습니다. 자세한 내용은 [BE 문서](BE_Readme.md#ai-service-통합)를 참고합니다.

## AI 완료 조건

- [x] 이미지 분석 실제 구현 (Bedrock/vLLM)
- [x] 답례 추천·실상품 검색 실제 구현
- [x] Google Calendar MCP 연결 및 확정 흐름
- [x] 경조사·영수증 등 추천 제외 대상 분기
- [ ] BE `AiRecommendationClient`를 `/api/v1/agent/recommend` 계약에 맞게 수정
- [ ] GPU 컨테이너 빌드와 모델 사전 적재(vLLM 경로를 실제로 쓸 경우)
- [ ] 동시 요청 부하 테스트
- [ ] Tavily credit 모니터링, 운영 API key 교체
