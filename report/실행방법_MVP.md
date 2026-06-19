# 실행 방법 — MVP 로컬 풀스택

> 앱 ↔ 서버 ↔ AI를 로컬에서 한 번에 띄우는 가이드. **순서: AI(8001) → 서버(8000) → 앱.** (AI가 먼저 떠야 서버가 프록시 가능)

## 0. 준비물
- **Python 3.11+**, **Node.js 20+**, **Flutter 3.24+**
- **키**: TourAPI 서비스키(공공데이터포털, ⚠️"일반 인증키(Decoding)") · Upstage LLM 키
- (선택) 카카오 키 — 앱 위치 좌표화, 현재 미사용

---

## 1. AI 서버 — dokkaebi-ai (포트 8001)
```bash
cd dokkaebi-ai
python3 -m venv .venv && source .venv/bin/activate     # (선택) 가상환경
pip install -r requirements.txt
cp .env.example .env
#  .env 에 채우기:
#    DOKKAEBI_TOURAPI_SERVICE_KEY=<디코딩 키>
#    DOKKAEBI_LLM_PROVIDER=upstage / DOKKAEBI_LLM_API_KEY=<키>
python3 -m uvicorn app.main:app --port 8001 --reload
#  → http://localhost:8001/v1/health  →  {"status":"ok"}
```
> 키 없어도 구동됨: LLM=mock 대사, TourAPI=mock 종로 노드. 단 실데이터·검색은 키 필요.

## 2. 게임 서버 — dokkaebi-server (포트 8000)
```bash
cd dokkaebi-server
npm install
npm run start:dev            # 또는: npm run build && node dist/main.js
#  → http://localhost:8000/v1/health
#  → Swagger UI: http://localhost:8000/docs
```

## 3. 앱 — dokkaebi-app
```bash
cd dokkaebi-app
flutter pub get
flutter run -d chrome        # 제일 쉬움 (또는 -d macos / 실기기)
```
→ 온보딩 → 게스트 로그인 → 메인(5탭). "새 탐험 시작"으로 코스 생성.

---

## 4. 앱 없이 빠르게 확인 — Swagger
브라우저 **http://localhost:8000/docs** → `POST /v1/scenarios/custom` → Try it out:
```json
{ "user_id":"u","start":{"lat":37.5703,"lng":126.9856},
  "transport":"walk","with_dialogue":true }
```
→ 종로 5조각 코스 + 도깨비 대사 JSON.

---

## 5. ⚠️ 자주 막히는 것
| 증상 | 원인·해결 |
|---|---|
| `AggregateError` / 생성 시 500 | **AI(8001)가 안 떠 있음.** AI 먼저 실행. (`curl localhost:8001/v1/health`) |
| `ERR_CONNECTION_REFUSED` | 서버(8000)가 안 떠 있음 |
| TourAPI 인증 오류 | "Encoding 키" 넣음 → **"Decoding 키"**로 |
| 에뮬레이터에서 서버 연결 안 됨 | `lib/config.dart`의 `localhost` → 안드로이드 에뮬 `10.0.2.2` / 실기기 `PC IP` |
| 검색·생성이 빈 결과 | 서버 2개 다 떠 있는지, TourAPI 키 들어갔는지 |

---

## 6. (선택) Docker 풀스택 — dokkaebi-infra
```bash
cd dokkaebi-infra
docker compose up --build      # ai + postgres + redis
```
> 배포·공유 캐시는 `DOKKAEBI_CACHE_BACKEND=redis`로.

---

## 부록 — 엔드포인트
**AI(8001)**: `/v1/health` · `/v1/scenarios` · `/v1/search` · `/v1/dialogue/turn` · `/v1/dialogue`
**서버(8000)**: `/v1/auth/guest` · `/v1/scenarios/custom` · `/v1/scenarios/search` · `/v1/dialogue/turn` · `/docs`

## 부록 — 실행 순서 한눈에
```
터미널1: AI 8001    →    터미널2: 서버 8000    →    터미널3: flutter run
        (먼저!)                                     (또는 /docs로 확인)
```
