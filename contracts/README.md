# API 계약 (Contracts) — 병렬 개발의 단일 소스

> 앱/서버/AI가 **서로를 mock으로** 두고 병렬 개발하기 위한 계약. 변경은 **PR로 공유**(계약 우선). 관리: 김예슬.

## 계약 3종

| 경계 | 계약 | 위치 |
|---|---|---|
| 앱 ↔ 게임서버 (REST) | `server-openapi.yaml` (OpenAPI 3.1) | 이 폴더 ✅ |
| 앱 ↔ 게임서버 (실시간) | WebSocket 이벤트 (아래 표) | 이 문서 |
| 게임서버 ↔ AI (내부) | `POST /v1/dialogue` (DialogueRequest/Response) | `dokkaebi-ai` (`app/api/schemas.py`) |

> 용어·스키마는 정본 [report/기획_통합.md](../report/기획_통합.md) (LocationQuest 13-A, 상태머신 3절) 기준. 필드는 snake_case.

## WebSocket / Socket.io 이벤트 (멀티 협력·경쟁)

> OpenAPI는 REST만 다뤄서 실시간 이벤트는 여기서 계약. 룸 = `party:{party_id}`.

| 이벤트 | 방향 | 페이로드 | 설명 |
|---|---|---|---|
| `party:join` | C→S | `{ code }` | 파티 입장 |
| `party:state` | S→C | `Party` | 파티 상태 동기화(멤버·위치·조각 수) |
| `fragment:collected` | S→C(broadcast) | `{ user_id, fragment_id }` | 누가 조각 획득(파티 전체 공유) |
| `chat:message` | C↔S | `{ user_id, text }` | 인게임 채팅 |
| `ranking:update` | S→C | `[{ user_id, score, rank }]` | 실시간 랭킹(경쟁) |
| `memory:restored` | S→C(broadcast) | `{ region_id }` | 합동 복원 연출 트리거 |

> 조각 획득의 **중복 방지(원자성)·랭킹**은 서버가 Redis로 처리(아키텍처 4절). 클라는 `fragment:collected` 수신만.

## mock으로 병렬 개발하기

```bash
# OpenAPI → mock 서버 (앱 팀이 서버 없이 개발)
npx @stoplight/prism-cli mock contracts/server-openapi.yaml   # http://localhost:4010

# OpenAPI → 타입/클라이언트 코드 생성 (앱: dart, 서버: ts 등)
npx @openapitools/openapi-generator-cli generate \
  -i contracts/server-openapi.yaml -g dart -o dokkaebi-app/lib/api
```

- **앱(정찬희)**: prism mock 또는 생성 클라이언트로 화면 먼저 개발
- **서버(김예슬)**: 이 계약대로 엔드포인트 구현, AI는 `dokkaebi-ai`로 프록시
- **AI(박준형/이지선)**: `dokkaebi-ai` 단독(노트북·`/v1/dialogue`)으로 검증 — server 불필요

## 변경 규칙

1. 계약 먼저 수정 → PR (제목 `[contract] ...`)
2. 머지 후 각 레포가 클라/타입 재생성
3. breaking change는 버전 올림(`info.version`) + 마이그레이션 노트
