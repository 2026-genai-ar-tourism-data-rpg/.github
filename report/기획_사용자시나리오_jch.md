# 시나리오 설계 — 도깨비: 팔도의 비밀

## 0. 문서 목적과 범위

- 대상: 한 관광지에서 **도착 → 보상**까지 굴러가는 게임플레이 플로우
(서사/세계관 문서와 별개)
- 산출: (1) 어느 장소에나 적용되는 **플로우 템플릿**,
(2) **NPC 합성 공식**, (3) **광화문 파일럿 예시**
- 정합 기준: 제안서(GPS 반경·contentTypeId·RAG NPC),
기획안(기억석·도감·방문률·유물, 탐험·수집 중심)
- 비범위: 실제 코드 구현, TourAPI contentId/좌표 바인딩(데이터 담당 P2/P3 영역, 본 문서는 TODO 슬롯만 정의)

핵심 게임플레이 1줄: **관광지 도착 → GPS 인증 → AR 카메라로 장소 도깨비 NPC 등장 → 대화·퀘스트 → 완료 → 보상**

---

## 1. 핵심 게임플레이 플로우 (상태 머신)

```
[ARRIVED]            관광지 트리거 반경 진입
   │  GPS 인증 (반경: 도심 50m / 개방공간 100m)
   ▼
[GPS_VERIFIED]       방문 인정 → 방문률 +, 탐험 경험치 +
   │  AR 카메라 실행
   ▼
[NPC_SPAWNED]        도깨비 NPC가 AR 오버레이로 등장
   │  최초 조우 시 도깨비 도감 등록
   ▼
[DIALOGUE]           NPC 대화(RAG 기반) → 퀘스트 수령
   ▼
[QUEST_ACTIVE]       퀘스트 타입별 수행 (1~2개 조합)
   │  성공 조건 충족
   ▼
[QUEST_COMPLETE]     완료 검증
   ▼
[REWARDED]           보상 지급: 경험치 / 기억석 조각 / 희귀 유물 / 도감
```

재방문 시: 퀘스트가 1회성으로 이미 완료된 장소는 `[NPC_SPAWNED]`에서 일상 대사 모드로 분기(2.6, 7절 참조).

---

## 2. NPC 합성 공식 (핵심)

목적: 임의의 TourAPI 관광지 1건 → 그 장소 고유 도깨비 NPC 1체를 **규칙으로** 생성. 수작업 없이 확장 가능해야 함(제안서 "확장 비용 낮음").

### 2.1 입력 (TourAPI)

- `title`, `contenttypeid`, `addr`, `overview`(장소 설명), `cat1/cat2/cat3`(카테고리)
- 대표 인물·상징은 `overview`/`title` 텍스트에서 추출

### 2.2 단계

1. **모티프 추출** — 우선순위로 1개 선정
    1. 대표 역사 인물: `overview`/`title`에 인물명 존재 (세종대왕, 이순신 등)
    2. 상징물·건축 요소: 현판, 해태상, 석탑 등
    3. 장소 기능: `contenttypeid` 기반 — 14/39(문화시설/음식점)=기능인, 32(숙박)=주인, 사찰=수도자, 시장=상인
    4. 자연·전설 요소: 바람, 용, 산신 등(제주·부산 권역)
2. **아키타입 매핑** — 모티프 → 4종 중 하나
    - `persona`(인물형): 실존 인물 모티프. 인물 업적·성격 반영
    - `guardian`(수호형): 건축·상징물 수호자. 위엄·과묵
    - `trade`(기능형): 상인/학예사/주인. 친근·수다
    - `spirit`(자연형): 자연·전설. 신비·변덕
3. **외형 합성** — 아키타입 base(뿔·방망이·한복 변형) + 모티프 시각 요소
    - 예: 세종 → 곤룡포 변형 + 작은 뿔 + 붓 모양 방망이
4. **persona/말투 생성** — 도깨비 공통 어미("~니라", "허허", "~겠느냐") + 모티프 색(세종=백성·글 강조, 자애)
5. **역할 고정** — 의뢰자(quest giver) 겸 장소 기억석 조각의 수호자
6. **대사 생성** — RAG로 `overview`/역사 주입 후 persona로 발화(5절 프롬프트)

### 2.3 출력

NPC 객체: `npc_id, name, archetype, motif, appearance_tags, persona, prompt_template_id, rag_source_ref(=tour_content_id)`

### 2.4 자동화 수준

- 초기(파일럿): 모티프 추출은 **큐레이션**(수동 지정)
- 확장기: `overview` → LLM 기반 인물/상징 추출 자동화로 전환
- 모티프 추출 실패(설명 빈약) 시 fallback: `contenttypeid` 기반 `trade`/`guardian` 기본값(7절 Edge Case)

---

## 3. 퀘스트 타입 시스템

기획안의 "추리·퍼즐보다 탐험·수집 중심" 기조에 따라, 무거운 추리가 아닌 **현장 탐색·수집·대화** 중심 4종으로 정의. 템플릿은 4종 모두 지원, 각 장소 인스턴스가 1~2개 선택.

```
QuestType:
  FIND_OBJECT    # AR 화면 속 숨은 오브젝트를 탭하여 수집
  PHOTO_MISSION  # 지정 대상이 화면에 들어오게 촬영 → 인증
  DIALOGUE       # NPC 질문에 선택지로 응답
  COLLECT        # 지정 AR 아이템 N개 획득
```

---

## 4. 데이터 스키마 — LocationQuest

장소 단위 퀘스트 1건의 데이터 구조. (실제 코드 구현 시 매직넘버는 상수화 — 별도 구현 단계)

```yaml
LocationQuest:
  quest_id: str
  tour_content_id: str          # TODO: TourAPI 바인딩 (데이터 담당)
  content_type_id: int          # 12 | 14 | 39 | 15 | 32
  coordinates:
    map_x: float                # TODO: TourAPI mapX
    map_y: float                # TODO: TourAPI mapY
  trigger_radius_m: int         # 50(도심) | 100(개방공간)
  density_tier: str             # "popular" | "low_traffic" -> reward_weight 결정

  npc:
    npc_id: str
    name: str
    archetype: str              # persona | guardian | trade | spirit
    motif: str
    appearance_tags: [str]
    persona: str
    rag_source_ref: str         # = tour_content_id
    prompt_template_id: str

  quest:
    types: [QuestType]          # 1~2개 조합
    objective: str
    success_condition: str
    hint: str

  reward:
    exp: int
    memory_stone_fragment_id: str
    rare_relic_id: str | null   # 기획안 유물 시스템 연계 (선택/시즌 게이트 가능)
    dex_entry: str              # 도깨비 도감 등록 키
```

> 분산 유도(제안서 핵심): `density_tier == "low_traffic"` 장소에 **최고 희귀도 조각**을 배치해 동선이 숨은 명소로 흐르게 한다. 종로의 경우 기획안에 4개 장소만 있으므로, 비인기 1슬롯(예: 운현궁/딜쿠샤/윤동주문학관 — 데이터 확인 후 확정)을 추가해 5개를 채우고 결정적 조각을 그곳에 둔다.
> 

---

## 5. RAG 프롬프트 템플릿 (`npc_dialogue_v1`)

```
[시스템]
너는 '{place_name}'을(를) 수호하는 도깨비 NPC '{npc_name}'다.
- 모티프: {motif}
- 아키타입: {archetype}
- 성격/말투: {persona}

[장소 실제 정보 — RAG 주입]
{tour_api_context}   # TourAPI overview, 역사·문화 설명

[규칙]
- 위 '장소 실제 정보'에 근거해서만 역사·문화를 말한다. 정보에 없으면 지어내지 않는다.
- 추리 유도가 아니라 '장소 소개 + 가벼운 힌트' 중심으로 말한다.
- 2~4문장. 도깨비 말투(어미 "~니라/~겠느냐", 감탄 "허허")를 유지한다.
- 사용자 진행 단계({stage})에 맞는 대사만 한다: 등장 / 의뢰 / 힌트 / 완료.

[컨텍스트]
- 사용자 진행상황: {player_state}
- 사용자 발화: {user_input}
```

placeholder는 런타임에 TourAPI·플레이어 상태로 치환. `{tour_api_context}`가 RAG 검색 결과 주입 지점.

---

## 6. 광화문 파일럿 예시

### 6.1 NPC — 글빛 도깨비

- `archetype`: persona (인물형)
- `motif`: 세종대왕 / 한글
- 외형: 곤룡포 변형 + 작은 뿔 + 붓 모양 방망이
- persona: 백성과 글을 아끼는 자애롭고 학구적인 도깨비 어조. 1인칭 "나", 어미 "~니라/~겠느냐"

### 6.2 퀘스트

- types: `DIALOGUE` + `FIND_OBJECT`
- objective: 광화문 광장 일대에 흩어진 '잊힌 한글 자모 조각' 5개를 AR로 찾아 모은다
- success_condition: 자모 조각 5개 수집
- hint: NPC가 위치 힌트 제공 ("현판을 올려다본 곳 근처…")
- (선택 PHOTO_MISSION 변형: 광화문 현판이 화면에 들어오게 촬영 인증)

### 6.3 보상

- 탐험 경험치
- 기억석 조각 (종로 기억석 1/5)
- 희귀 유물: **조선 왕실 인장 복제품** (기획안 종로 유물과 일치) — 역사 퀘스트 보상 +10% 버프
- 도감 등록: 글빛 도깨비

### 6.4 NPC 대사 샘플 (단계별)

- **등장**: "허허, 마침내 나를 볼 수 있는 눈을 가진 이가 왔구나. 나는 이 문을 오래도록 지켜온 도깨비니라."
- **의뢰**: "백성들이 글자를 잊으니 이 일대의 기억도 함께 흐려진다. 흩어진 글자 조각을 나와 함께 찾아주겠느냐?"
- **힌트(진행 중)**: "현판의 글씨를 올려다본 적 있느냐. 그 시선이 닿은 곳을 살펴보거라."
- **완료**: "장하다, 탐사자여. 잊혔던 글빛이 다시 돌아오는구나. 이 인장을 받거라. 이 땅의 기억을 지킨 증표이니라."

### 6.5 스키마 인스턴스

```yaml
quest_id: "jongno_gwanghwamun_01"
tour_content_id: "TODO_TOURAPI_BIND"     # TODO: TourAPI 광화문 contentId
content_type_id: 12
coordinates:
  map_x: 0.0                              # TODO: TourAPI mapX
  map_y: 0.0                              # TODO: TourAPI mapY
trigger_radius_m: 100                     # 광장 = 개방공간
density_tier: "popular"
npc:
  npc_id: "npc_gwanghwamun_sejong"
  name: "글빛 도깨비"
  archetype: "persona"
  motif: "세종대왕/한글"
  appearance_tags: ["곤룡포변형", "작은뿔", "붓방망이"]
  persona: "백성과 글을 아끼는 자애롭고 학구적인 도깨비 어조"
  rag_source_ref: "TODO_TOURAPI_BIND"
  prompt_template_id: "npc_dialogue_v1"
quest:
  types: ["DIALOGUE", "FIND_OBJECT"]
  objective: "흩어진 한글 자모 조각 5개 수집"
  success_condition: "collected_jamo == 5"
  hint: "현판 주변 탐색"
reward:
  exp: 100
  memory_stone_fragment_id: "jongno_stone_1of5"
  rare_relic_id: "relic_royal_seal"
  dex_entry: "글빛 도깨비"
```

---

## 7. Edge Case

- **GPS 위조/오차**: 실제 이동 없는 위치 스푸핑 → 정확도 임계값·이동 속도 이상 탐지로 인증 신뢰도 확보
- **실내·음영 지역 GPS 부정확**: AR 인식 보조 인증 또는 반경 완화 정책
- **모티프 추출 실패**(`overview` 빈약): `contenttypeid` 기반 `trade`/`guardian` 기본 아키타입으로 fallback
- **재방문 중복**: 1회성 퀘스트 완료 후 NPC는 일상 대사 모드로 전환(신규 퀘스트 미발급)
- **멀티 동시성**: 4인이 같은 AR 오브젝트 탭 → 조각 중복/분배 동기화 규칙 필요
- **사진 미션 검증**: 정답 판정 기준(온디바이스 분류 신뢰도, 오인식 허용 범위)
- **시즌 퀘스트**(contentTypeId 15): 행사 기간 종료 후 NPC 등장·퀘스트 만료 처리, 기간 데이터 부재 시 fallback
- **AR 미지원/저사양 기기**: 2D 대체 모드 등 fallback 경로
- **비인기 핵심 장소 접근 불가**(영업 종료 등): 결정적 조각 회수 차단 → 대체 단서 또는 우회 조건

---

## 8. 확장 적용 (다음 단계)

- 같은 템플릿으로 종로 나머지 4슬롯(인기) + 비인기 1슬롯 채워 종로 챕터 완성
- 이후 경주·전주·부산·제주는 NPC 합성 공식만 재적용 → 콘텐츠 제작 비용 최소화
- 데이터 담당과 분리: 본 문서의 모든 `TODO_TOURAPI_BIND`/좌표는 TourAPI 연동 시 채워짐