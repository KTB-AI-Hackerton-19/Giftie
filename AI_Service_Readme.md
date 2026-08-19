# Giftie AI Service

[전체 기획서로 돌아가기](README.md)

실제 실행법, 전체 응답 예시, 함수 시그니처와 구현 지점은
`../AI-Service/README.md`에 가장 상세하게 정리되어 있습니다.

## 역할

AI-Service는 Spring Boot가 전달한 직접 선물데이터 또는 이미지 URL을 공통
`GiftData`로 만들고, 네 독립 작업을 비동기로 실행해 하나의 JSON으로 반환하는
FastAPI 오케스트레이터입니다.

## 공개 API

| Method | Path | 역할 |
|---|---|---|
| POST | `/api/v1/agent/from-gift-data` | 구조화된 선물데이터로 네 작업 실행 |
| POST | `/api/v1/agent/from-image` | 이미지 분석 후 네 작업 실행 |

두 API 모두 `X-API-KEY`가 필요합니다.

## 처리 구조

```text
직접 GiftData ───────────────────────────┐
                                        ├─ 공통 GiftData
이미지 URL → 이미지 분석(mock) ─────────┘
                   │
                   ├─ 선물 기록 데이터 준비(mock)
                   ├─ 캘린더 데이터 준비(mock)
                   ├─ 알림 데이터 준비(mock)
                   └─ Qwen 추천·메시지
                          └─ Tavily 카테고리별 실상품 검색
                   │
                   └─ 비동기 결과 병합 → JSON
```

최상위 네 작업은 `asyncio.gather(..., return_exceptions=True)`로 동시에 실행합니다.
한 작업이 실패해도 나머지 결과는 유지하고 실패한 작업만 `ERROR`로 반환합니다.

## 실제 구현과 mock 구분

| 기능 | 상태 |
|---|---|
| 이미지에서 GiftData 추출 | mock, 담당자 구현 필요 |
| 선물 기록 저장용 JSON | mock 시그니처 준비 |
| Google Calendar MCP 데이터 | mock 시그니처 준비 |
| 알림 예약 데이터 | mock 시그니처 준비 |
| Qwen 추천 카테고리·이유·메시지 | 실제 실행 |
| Tavily 실상품 검색 | 실제 실행 |
| 입력 정규화와 예외 처리 | 실제 실행 |

## 추천 정책

- 받은 가격의 80~120%를 기본 범위로 사용
- 금액대별 100원, 1,000원, 10,000원 단위 반올림
- `age`가 `0`, `"0"`, 빈 문자열 또는 `null`이면 미입력 처리
- 관계·이름의 빈 문자열과 공백은 미입력 처리
- 잘못된 선택 날짜는 미입력 처리
- 카테고리 whitelist와 점수 범위 검증
- 모델 JSON 파싱 실패 시 재시도와 fallback

## 실상품 검색 정책

- Qwen이 카테고리별 검색어 생성
- Tavily 검색은 카테고리별 병렬 실행
- 허용된 쇼핑몰만 통과
- 검색 목록이 아닌 개별 상품 URL만 통과
- 상품명과 추천 카테고리의 의미 관련성 검증
- 가격을 읽을 수 있는 상품만 사용
- 범위 안 상품 우선
- 부족하면 가장 가까운 가격의 관련 상품을 `NEAREST`로 반환
- 검색 실패 시 `products: []`, 기존 `product_examples` 유지

## 로컬 실행

Apple Silicon에서는 MLX를 사용합니다.

```bash
cd AI-Service
source .venv-runtime/bin/activate
python -m uvicorn app.main:app --host 0.0.0.0 --port 8999
```

- Swagger: `http://127.0.0.1:8999/docs`
- 테스트: `pytest -q`

## 주요 환경변수

```env
API_KEY=
MODEL_BACKEND=mlx
LOCAL_MODEL_ID=mlx-community/Qwen3-4B-Instruct-2507-4bit
PRELOAD_MODEL=false
REQUEST_TIMEOUT_SECONDS=45
PRODUCT_SEARCH_PROVIDER=auto
TAVILY_API_KEY=
PRODUCT_SEARCH_TIMEOUT_SECONDS=8
```

## GPU 서버 배포

- NVIDIA GPU 서버에서는 `MODEL_BACKEND=transformers`를 사용합니다.
- Qwen 모델은 서버 시작 시 미리 적재하는 것을 권장합니다.
- GPU당 Uvicorn worker 1개를 기본으로 합니다.
- AI-Service는 public endpoint 대신 private subnet에 두고 BE에서만 호출합니다.
- BE와 같은 `AI_SERVICE_API_KEY`를 Secrets Manager로 주입합니다.
- 일반 JSON을 HTTPS로 전송하며 별도 JSON 암호화는 적용하지 않습니다.

## 담당자가 수정할 파일

| 작업 | AI-Service 파일 |
|---|---|
| 이미지 분석 | `app/services/tasks/image_analysis.py` |
| 선물 기록 데이터 | `app/services/tasks/gift_record.py` |
| 캘린더 데이터 | `app/services/tasks/calendar.py` |
| 알림 데이터 | `app/services/tasks/notification.py` |

각 파일의 `IMPLEMENTATION POINT` 주석과 함수 시그니처를 유지하면서 내부 구현을
교체해야 합니다.

## AI 완료 조건

- [ ] 이미지 분석 실제 구현
- [ ] BE 최종 DTO 계약 고정
- [ ] Google Calendar MCP 연결 정책 확정
- [ ] 알림 데이터 BE 계약 반영
- [ ] GPU 컨테이너 빌드와 모델 사전 적재
- [ ] 동시 요청 부하 테스트
- [ ] Tavily credit·timeout 모니터링
- [ ] 운영 API key 교체

