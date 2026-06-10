# dokkaebi-infra

> 전체 서비스의 배포·운영 설정을 모은 단일 진실 공급원(SSOT).

소규모 팀이 한 번의 명령으로 풀스택을 띄울 수 있도록 인프라 설정을 한곳에 모읍니다.

## Clone

```bash
git clone https://github.com/2026-genai-ar-tourism-data-rpg/dokkaebi-infra.git
cd dokkaebi-infra
```

## 내용

- `docker-compose.yml` — 로컬 풀스택 한 번에 기동 (server + ai + postgres + redis)
- 배포 매니페스트 — **핫패스 = 컨테이너**(ECS Fargate/EC2), **배치 = 서버리스**(Lambda/스케줄)
- GitHub Actions **재사용 워크플로우** (각 레포에서 호출)
- 환경변수·시크릿 템플릿 (`.env.example`)
- DB 마이그레이션 정책

## 배포 모델 (요약)

> 상세: [report/아키텍처_파이프라인.md](../report/아키텍처_파이프라인.md) 9절

- **Persistent 컨테이너**: `dokkaebi-server`(실시간·WebSocket) + `dokkaebi-ai`(LLM 오케스트레이션·지역 인메모리) — 상태/지속연결/워밍 캐시 때문에 Lambda ❌
- **Serverless(Lambda/스케줄)**: 데이터 배치(수집·노드빌더·임베딩)·온디맨드 fetch·스파이크성 stateless 엔드포인트. 무거운 배치는 AWS Batch/Fargate Task
- **MVP**: 핫패스 컨테이너 + 배치만 Lambda. 풀 서버리스 ❌

## 로컬 풀스택 실행

```bash
cp .env.example .env   # 값 채우기 (TourAPI 키, LLM 키 등)
docker compose up
```

> `docker compose up` 한 번으로 server·ai·db·redis가 함께 뜨도록 구성해 온보딩·로컬 테스트 비용을 최소화합니다.

## 관련 레포

- [`dokkaebi-server`](./dokkaebi-server.md)
- [`dokkaebi-ai`](./dokkaebi-ai.md)
