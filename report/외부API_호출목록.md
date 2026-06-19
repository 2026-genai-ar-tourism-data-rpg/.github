# 외부 API 호출 목록 — 어떤 상황에 무엇을 부르나

> 도깨비가 호출하는 **TourAPI(한국관광공사) + 외부 서비스**를 **상황별**로 정리. 구현 상태(✅실동작 / 🟡골격 / ⬜계획)도 표기. 코드 = `dokkaebi-ai/app/tourapi/`·`app/embeddings/`·`app/llm/`.
>
> 큰 원칙: **TourAPI는 "노드 만들 때/생성할 때"만 부르고, 받은 건 우리 DB에 저장(persist-on-touch). 플레이 중엔 거의 안 부름.** (상세 [아키텍처](./아키텍처_파이프라인.md) 1-1·5-5)

---

## 1. 한눈에 — 상황별 호출

| 상황 | 무엇을 | 서비스 / 오퍼레이션 | 상태 |
|---|---|---|---|
| **입력: 가고싶은 곳 검색**(자동완성) | 관광지 이름 → 후보 | `KorService2 / searchKeyword2` | ✅ |
| **입력: 위치 좌표화**(현재/집) | 주소·장소명 → 좌표 | 카카오 로컬검색 / 폰 GPS | ⬜(앱) |
| **생성: 반경 후보** | 좌표+반경 → 거리순 관광지 | `KorService2 / locationBasedList2` | ✅ |
| **생성: 설명 텍스트**(대사 재료) | contentId → overview | `KorService2 / detailCommon2` | ✅ |
| **생성: NPC 대사** | overview → 도깨비 대사 | LLM (Upstage Solar) | ✅ |
| **노드 워밍/적재**(배치) | 시군구 관광지 전체 | `KorService2 / areaBasedList2` | ⬜ |
| **비인기 라벨링**(나중) | 집중률·중심성·연관 | 빅데이터 4종 (아래 3절) | ✅(접근층) |
| **의미검색**(나중) | 자유문장 → 비슷한 노드 | 임베딩 (Upstage) | 🟡(mock) |
| **영업시간·요금**(나중) | contentId → 이용정보 | `KorService2 / detailIntro2` | ⬜ |
| **이미지**(나중) | contentId → 사진 | `KorService2 / detailImage2` | ⬜ |
| **플레이 중** | — | (TourAPI 안 부름, 우리 DB·캐시만) | — |

---

## 2. KorService2 — 노드 본문(좌표·텍스트). **MVP 핵심**

> 한 곳의 **기본 정보·설명·이미지**를 주는 서비스. 베이스 `http://apis.data.go.kr/B551011/KorService2`.

| 오퍼레이션 | 무엇을 주나 | 어떤 상황 | 코드 | 상태 |
|---|---|---|---|---|
| `searchKeyword2` | 이름으로 관광지 검색(좌표·contentId) | **앵커 자동완성**(입력) | `client.search_keyword` | ✅ |
| `locationBasedList2` | 좌표+radius 내 관광지(거리순·dist) | **반경 후보**(생성) | `client.location_based_list` | ✅ |
| `detailCommon2` | overview 설명 텍스트(homepage·tel) | **대사 grounding 재료**(생성) | `client.detail_common` | ✅ |
| `areaBasedList2` | 시군구 관광지 목록 | **노드 워밍**(빌드타임 배치) | — | ⬜ |
| `detailIntro2` | 이용요금·영업시간·휴무 | 예산·영업시간(나중) | — | ⬜ |
| `detailImage2` | 관광지 사진 URL | 에셋 보강(나중) | — | ⬜ |

- ⚠️ **`searchKeyword2`는 부분일치** → "경복궁" 검색 시 "한복남 경복궁점"도 옴 → **앱 자동완성에서 사용자가 탭**(정확 title 우선). 지역코드 필터 ❌(정답 누락).
- **contentId = 그 장소 고유 ID**(경복궁=126508). 이걸로 `detailCommon2` → 대사 재료. ↔ contentTypeId(12=관광지)는 유형 필터.

---

## 3. 빅데이터 4종 — 비인기/연관(통계). **사용은 나중(박준형 EDA)**

> 통계·랭킹·연관을 주지만 **좌표·설명은 없음**(코드도 KorService2와 다름 → 이름/좌표 매칭 필요). 접근 코드는 완성(`bigdata.py`), 사용 로직은 비인기 라벨링 task에서.

| 서비스 / 오퍼레이션 | 무엇을 주나 | 어떤 상황 | 코드 | 상태 |
|---|---|---|---|---|
| `TarRlteTarService1 / areaBasedList1·searchKeyword1` | 함께 방문되는 연관 관광지 | 시나리오 **동선 후보**(나중) | `bigdata.related_by_area/keyword` | ✅ |
| `TatsCnctrRateService / tatsCnctrRatedList` | 향후 30일 집중률(혼잡도) | **비인기/한산도 라벨** + 덜 붐비는 날 | `bigdata.concentration_rate` | ✅ |
| `LocgoHubTarService1 / areaBasedList1` | 시군구 중심 관광지 랭킹(좌표 포함) | **인기/앵커 vs 비인기/샛길** | `bigdata.hub_attractions` | ✅ |
| `DataLabService / metco·locgoRegnVisitrDDList` | 지역 방문자수(일별) | 지역 트렌드 EDA | `bigdata.metco/locgo_visitors` | ✅ |

---

## 4. 오디오가이드 — 대본 텍스트(grounding 보강). **스펙 대기**

| 서비스 | 무엇을 | 상태 |
|---|---|---|
| (서비스명 미확정) | 관광지 오디오가이드 대본 → 긴 grounding 텍스트 | 🟡 골격만 (`audioguide.py`, 스펙 확인 후 채움) |

---

## 5. 비-TourAPI 외부 서비스

| 서비스 | 무엇을 | 어떤 상황 | 담당 | 상태 |
|---|---|---|---|---|
| **카카오 로컬검색/주소** | 주소·장소명 → 좌표 | 현재위치 외 위치 입력 | 정찬희(앱) | ⬜ |
| **카카오 길찾기**(Mobility) | 실제 도보 경로·거리 | 동선 정밀화(발전) | — | ⬜ |
| **폰 GPS**(geolocator) | 현재 위치 좌표 | 현재위치·도착 인증 | 정찬희(앱) | ⬜ |
| **LLM**(Upstage Solar `solar-pro`) | NPC 대사·연결 스토리 | 생성·대화 | 김예슬·박준형 | ✅ |
| **임베딩**(Upstage `solar-embedding`) | 텍스트 → 벡터 | 의미검색(나중) | 이지선·박준형 | 🟡(mock) |

---

## 6. TourAPI 공통 호출 규칙 (전 서비스 공통, `base.py`가 처리)

- **공통 파라미터**: `serviceKey`·`MobileOS=ETC`·`MobileApp`·`_type=json` (`base._common_params`)
- ⚠️ **serviceKey = "일반 인증키(Decoding)"** 사용 (httpx가 자동 인코딩 → Encoding 키 넣으면 이중인코딩 오류)
- ⚠️ **HTTP 200 ≠ 성공** → `response.header.resultCode`(0000) 확인 필수 (`base._unwrap`)
- **페이지네이션**: `totalCount`/`numOfRows`로 반복 (`base.request_all`, max_pages 상한)
- **키 없으면**: KorService2는 **mock 종로 노드**로 구동(`mock_nodes.py`) / 빅데이터는 키 필요

---

## 7. 구현 상태 요약

- ✅ **실동작**: searchKeyword2 · locationBasedList2 · detailCommon2 · 빅데이터 4종 · LLM 대사
- 🟡 **골격**: 오디오가이드(스펙 대기) · 임베딩(mock)
- ⬜ **계획**: areaBasedList2(워밍) · detailIntro2/Image2 · 카카오(앱) · 임베딩 실키

> 키: TourAPI ✅(발급됨) / 카카오·임베딩 ⬜(미발급). 카카오는 앱 쪽이라 AI는 좌표만 받으면 됨.
