# DB 스키마 v0 — 도깨비

> 3층 모델([아키텍처](./아키텍처_파이프라인.md) 5-4) 기준. **노드(장소)=1번 저장 / 시나리오=node_id 참조 / 세션=실시간은 Redis**.
> 담당: 설계 = 김예슬·이지선 / 게임 데이터 함수 = 이지선·정찬희. v0이므로 구현하며 컬럼 조정 가능.

---

## 🟢 PostgreSQL (영구 — 게임이 신뢰하는 사본)

### `nodes` — 관광지 노드 (TourAPI에서 받아 복사, persist-on-touch)
```sql
CREATE TABLE nodes (
  node_id          TEXT PRIMARY KEY,          -- 예: 'jongno_unhyeongung'
  name             TEXT NOT NULL,
  region           TEXT NOT NULL,             -- '종로'
  content_type_id  INT  NOT NULL,             -- 12=관광지 (식당39 등은 나중)
  map_x            DOUBLE PRECISION NOT NULL, -- 경도
  map_y            DOUBLE PRECISION NOT NULL, -- 위도
  addr1            TEXT, addr2 TEXT,
  category         TEXT,                      -- '고궁·역사' 등
  trigger_radius_m INT  NOT NULL DEFAULT 50,  -- GPS 인증 반경(50/100/150)
  density_tier     TEXT NOT NULL,             -- 'popular' | 'low_traffic'
  overview         TEXT,                      -- 대화 grounding 텍스트
  audio_guide_text TEXT,                      -- 오디오가이드 대본
  npc              JSONB,                     -- {npc_id, archetype, motif, persona, image_url}
  tour_content_id  TEXT,                      -- TourAPI 바인딩 키
  source           TEXT DEFAULT 'TourAPI',
  fetched_at       TIMESTAMPTZ,
  updated_at       TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_nodes_region        ON nodes(region);
CREATE INDEX idx_nodes_density       ON nodes(region, density_tier);  -- 샛길(비인기) 선택용
CREATE INDEX idx_nodes_coords        ON nodes(map_x, map_y);          -- 반경 후보(필요 시 PostGIS)
```

### `scenarios` — 시나리오 정의 (루트). 공용 메타도 여기
```sql
CREATE TABLE scenarios (
  scenario_id      TEXT PRIMARY KEY,          -- 'scn_jongno_001'
  title            TEXT NOT NULL,
  region           TEXT NOT NULL,
  constraints      JSONB,                     -- {time:'2h', transport:'walk', ...}
  story            TEXT,                      -- LLM이 쓴 연결 스토리
  type             TEXT NOT NULL,             -- 'official' | 'custom' | 'public'
  created_by       TEXT REFERENCES users(user_id),
  forked_from      TEXT REFERENCES scenarios(scenario_id),  -- 리믹스 계보
  -- 공용 풀 메타 (승격 시 채움)
  status           TEXT DEFAULT 'private',    -- 'private' | 'public'
  rating           REAL DEFAULT 0,
  completion_rate  REAL DEFAULT 0,
  plays            INT  DEFAULT 0,
  created_at       TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_scenarios_region_status ON scenarios(region, status);  -- 재사용 풀 조회
```

### `scenario_nodes` — 시나리오 ↔ 노드 (순서 있는 참조, 정규화)
```sql
CREATE TABLE scenario_nodes (
  scenario_id   TEXT NOT NULL REFERENCES scenarios(scenario_id) ON DELETE CASCADE,
  order_index   INT  NOT NULL,                -- 동선 순서 0,1,2…
  node_id       TEXT NOT NULL REFERENCES nodes(node_id),  -- 본문 복사 X, 참조만
  role          TEXT NOT NULL,               -- 'anchor' | 'side' | 'bonus'
  fragment_slot TEXT,                         -- '1/5'
  reward_tier   TEXT,                         -- 'rare' 등
  grants        JSONB,                        -- 연계: 이 노드 완료 시 획득(조각·단서·플래그) (7-C)
  requires      JSONB,                        -- 연계: 진입/완료 전제(필요 단서 등) (7-C)
  PRIMARY KEY (scenario_id, order_index)
);
```

### `users` — 유저
```sql
CREATE TABLE users (
  user_id        TEXT PRIMARY KEY,
  nickname       TEXT NOT NULL,
  explorer_grade INT  DEFAULT 1,              -- 탐사 등급(레벨)
  exp            INT  DEFAULT 0,
  created_at     TIMESTAMPTZ DEFAULT now()
);
```

### `user_visits` / `user_fragments` / `user_dex` — 개인 진행 기록
```sql
-- 방문(방문률 계산)
CREATE TABLE user_visits (
  user_id    TEXT REFERENCES users(user_id),
  node_id    TEXT REFERENCES nodes(node_id),
  visited_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (user_id, node_id)
);
-- 수집한 기억석 조각
CREATE TABLE user_fragments (
  user_id      TEXT REFERENCES users(user_id),
  fragment_id  TEXT NOT NULL,                 -- 'jongno_stone_1of5'
  scenario_id  TEXT REFERENCES scenarios(scenario_id),
  collected_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (user_id, fragment_id)
);
-- 도깨비 도감
CREATE TABLE user_dex (
  user_id     TEXT REFERENCES users(user_id),
  npc_id      TEXT NOT NULL,
  unlocked_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (user_id, npc_id)
);
```

### `scenario_runs` — 플레이 세션(완주 기록). 실시간 진행은 Redis
```sql
CREATE TABLE scenario_runs (
  run_id       TEXT PRIMARY KEY,
  scenario_id  TEXT REFERENCES scenarios(scenario_id),
  user_id      TEXT REFERENCES users(user_id),
  party_id     TEXT,                          -- 멀티면 파티 ID (룸 상태는 Redis)
  state        TEXT,                          -- ARRIVED|GPS_VERIFIED|QUEST_ACTIVE|COMPLETE|REWARDED
  progress     INT DEFAULT 0,                 -- 모은 조각 수
  inventory    JSONB,                         -- 연계: 누적 단서·조각·선택 플래그 (7-C, 실시간은 Redis)
  started_at   TIMESTAMPTZ DEFAULT now(),
  completed_at TIMESTAMPTZ
);
```

---

## 🖼️ 에셋(이미지·AR 오브젝트) 저장 — DB엔 URL만

> **원칙: 이미지·3D 모델 같은 바이너리는 PostgreSQL에 직접 넣지 않는다.** 오브젝트 스토리지(S3·CloudFront 등)에 파일을 두고, **DB에는 URL/키만** 저장. (DB 비대화·속도 저하·CDN 캐싱 때문)

- **NPC(도깨비) 이미지**: `nodes.npc.image_url` 에 보관 (NPC는 노드당 1체 — 8-B). 예: `"image_url": "https://cdn.dokkaebi/npc/dokkaebi_unhyeon.png"`
- **공유 에셋**(기억석 조각·유물·도감 일러스트·AR 3D 오브젝트)은 재사용되므로 별도 레지스트리:

```sql
CREATE TABLE assets (
  asset_id   TEXT PRIMARY KEY,            -- 'frag_memory_stone_default'
  type       TEXT NOT NULL,              -- 'npc' | 'fragment' | 'relic' | 'dex_art' | 'ar_object'
  url        TEXT NOT NULL,              -- 오브젝트 스토리지 URL
  format     TEXT,                       -- 'png' | 'glb'(3D) | 'usdz'(ARKit) 등
  meta       JSONB,                      -- 크기·라이선스·작가(이지선) 등
  created_at TIMESTAMPTZ DEFAULT now()
);
```
- 노드/유물/도감이 에셋을 쓸 땐 **`asset_id` 또는 URL을 참조**(노드처럼 1번 등록 → 여러 곳 재사용).
- AR 3D 오브젝트는 플랫폼별 포맷 주의: **Android(ARCore)=glb / iOS(ARKit)=usdz** → `assets`에 둘 다 등록하거나 `meta`로 분기.
- 담당: 에셋 제작·등록 = **이지선**(디자인), 앱에서 URL 로드·AR 렌더 = **정찬희**.

---

## 🔴 Redis (임시·실시간 — 키 패턴)

| 키 | 타입 | 내용 | 비고 |
|---|---|---|---|
| `region:{region}:nodes` | Hash | 지역 워킹셋(노드 텍스트) | LRU evict, 원천=`nodes` |
| `party:{roomId}` | Hash | 파티방(code·members·scenario) | 최대 4인 |
| `party:{roomId}:fragments` | Hash | 조각 선점(누가 먼저) | **Lua 원자처리**(중복 방지) |
| `ranking:{region}` | Sorted Set | 점수순 랭킹 | ZADD/ZRANGE |
| `dialogue:{nodeId}:{stage}` | String | NPC 대사 캐시 | TTL, 미스 시 LLM |
| `session:{userId}` | String | 세션 토큰 | TTL |

---

## 관계 한눈에

```
users ─┬─< user_visits >─ nodes
       ├─< user_fragments
       ├─< user_dex
       └─< scenarios(created_by)
scenarios ─< scenario_nodes >─ nodes      ← 시나리오는 node_id로 '참조'만(정규화)
scenarios ─< scenario_runs >─ users
```

## v0 메모 / 후속 결정
- **반경 후보 조회**: 초기엔 `map_x/map_y` btree + 박스 필터로 충분. 정밀해지면 **PostGIS**(`geography`+GiST) 도입 — 발전.
- **임베딩 저장**: Tier1은 인메모리 numpy(런타임)라 DB 컬럼 불필요. pgvector 가면 `nodes.embedding vector(1024)` 추가.
- **맞춤 시나리오 보관 정책**(전부 영구 vs 승격분만)·**캐시 무효화**는 [기획 §17] 미결.
- ORM(TypeORM/Prisma) 선택 후 이 DDL을 엔티티/마이그레이션으로 옮김 (dokkaebi-server).
