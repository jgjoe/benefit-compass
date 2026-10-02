# 혜택나침반 (BenefitCompass)

**온통청년·정부24의 공식 정책 13,589건을 한 경로에서 검색하고, 검색된 정책만 근거로 답하는 RAG 서비스**

[![Live](https://img.shields.io/badge/live-demo-success)](https://jgjoe.github.io/benefit-compass)
[![Stack](https://img.shields.io/badge/stack-Spring%20Boot%20%2B%20FastAPI%20%2B%20pgvector-informational)](#아키텍처)
[![CI](https://img.shields.io/badge/CI-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)](.github/workflows)

정책과 혜택은 여러 공식 출처에 흩어져 있어 **정작 내가 받을 수 있는 게 뭔지 찾기 어렵습니다.**
나이와 "월세 지원 받고 싶어" 같은 질문을 넣으면 관련 정책을 찾고, **검색된 공식 정책만 근거로** 답합니다.

개인 프로젝트로 데이터 수집·정제부터 임베딩·벡터검색·답변 생성, 배포와 운영 검증까지 설계·구현·검증했습니다.
온통청년 2,631건과 정부24 10,958건을 같은 검색·답변 경로에 통합했고, 현재 검증된 corpus는 **정책 13,589건 / 청크 17,609개 / 임베딩 누락 0건**입니다.
후보 검색·리랭킹·지역 검색은 각각 평가했지만, 근거가 부족하거나 다른 필수 지표를 악화시킨 변경은 production에 넣지 않았습니다.

**[라이브 데모](https://jgjoe.github.io/benefit-compass)** — 요청이 없으면 인스턴스를 내리는 구성이라, 첫 요청은 서버와 모델을 올리는 동안 잠시 기다립니다.

![월세 지원 질문의 실제 검색 결과](docs/images/search-result.png)

---

## 주요 기능

- 나이와 "월세 지원 받고 싶어" 같은 질문으로 온통청년·정부24 공식 정책 13,589건을 한 번에 검색
- 검색된 공식 정책만 근거로 답하고 정책명을 함께 인용
- 요청 ID와 Prometheus 지표로 배포 후 지연·무결과를 관측

## 설계 판단

### RAG를 한 덩어리로 두지 않았다

검색 결과를 LLM에 통째로 넘기면 품질이 나빠졌을 때 **어디가 문제인지 알 수 없습니다.**
임베딩 · 벡터검색 · 선택적 리랭킹 · 생성을 단계로 쪼개 두니 단계별로 따로 측정할 수 있었고,
리랭킹이 어떤 지표를 개선하고 어떤 지표를 악화시키는지 분리해 판단할 수 있었습니다.

### 답변은 검색된 정책만 근거로 쓴다

모델이 정책을 지어내면 사용자가 존재하지 않는 지원금을 신청하러 갑니다.
검색 결과 범위 안에서만 답하게 하고 정책명을 함께 인용하게 했습니다.
마땅한 결과가 없으면 "없다"고 단정하지 않고 다시 검색해 보도록 안내합니다.

### 지역 필터는 데이터를 믿을 수 없어 노출을 끊었다

`zipCd`를 법정동코드로 적재해 검색 SQL에 지역 조건까지 걸었는데, **서울로 필터링해도 함안군 정책이 통과**했습니다.
진단 스크립트(`ingest/inspect_region.py`)로 원본을 덤프해 보니 지자체 정책에 타지역 코드가 섞여 있었고
기관명도 부서명뿐인 경우가 많았습니다. 기관명 기반 보강 필터를 덧대 봤지만 **원본이 틀린 이상 신뢰할 수 없다고 판단**해
사용자 노출에서 제외했습니다.

**기능은 지우지 않고 노출만 끊었습니다.** 지역 필터 코드는 남겨 두어 원본 데이터를 정제하면 CLI(`ingest/search.py --region`)로 바로 다시 검증할 수 있고,
HTTP API는 `region`이 들어오면 `400 INVALID_REQUEST`로 거절해 믿을 수 없는 필터가 조용히 통과한 것처럼 보이지 않게 했습니다.
질의에 섞인 지역어는 검색 잡음이 되므로 `strip_region`으로 제거합니다.

### 임베딩은 외부 API 대신 로컬 모델

처음엔 Gemini 임베딩을 쓰려 했으나 무료 티어가 분당·일일 한도에 금방 걸렸습니다.
한국어에 강한 `multilingual-e5-base`를 컨테이너에서 직접 돌려 **모델 API 호출 한도를 제거**했습니다.

### API는 Spring Boot, ML은 Python

ML 라이브러리는 Python 생태계가 편하고 비즈니스 로직은 Spring Boot가 낫습니다.
둘을 한 프로세스에 두지 않고 서비스로 나눴습니다.

### 배포 뒤에도 볼 수 있게 했다

배포하고 끝내지 않고 **볼 수 있게** 만들었습니다.

- `/actuator/prometheus` — endpoint·상태 구간별 지연시간, 검색 결과/무결과 수집
- `/actuator/health` — 배포 상태 확인
- 모든 API 응답에 `X-Request-ID`를 넣어 장애 로그를 추적
- **질문 원문과 나이는 로그·메트릭에 저장하지 않습니다** (장애 조사 목적으로도 남기지 않음)

### 콜드스타트를 구간으로 분해했다

요청이 없으면 인스턴스를 내리는 구성에서 첫 요청 지연을 **체감이 아니라 구간별 수치로** 확인했습니다.

1. 요청 ID와 구간 header를 넣어 콜드 경로를 **API↔ML / 모델 준비 / 임베딩 / DB 연결·쿼리 / 생성**으로 분해
2. 공개 traffic을 바꾸지 않은 **0% revision**에서, 15분 유휴 뒤 before/after를 동시 호출하는 절차로 **5쌍 반복 측정**
3. 지배 구간은 `api_ml_transport` 중앙값 **26.9초** — ML scale-from-zero + 모델 준비 대기 + 큐. 모델 로딩만 23~24초
4. 비용이 들지 않는 조치를 적용: `/ready` startup probe로 모델 준비 전 트래픽 차단, 런타임 모델 허브 의존 제거

> 측정 원자료와 운영 문서: [Production Lab 2](docs/operations/PRODUCTION_LAB_2_2026-07-21.md) ·
> [운영 기준선](docs/operations/BASELINE_2026-07-14.md) · [SLO 초안](docs/operations/SLO.md) · [런북](docs/operations/RUNBOOK.md)

## 검증 결과

### 검색 품질은 측정한 뒤 채택했습니다

RAG는 답변이 그럴듯해 보여도 검색 단계가 틀릴 수 있습니다. 직접 라벨링한 Youth 60문항과 Gov24 21문항으로
production과 같은 검색 조건을 비교하고, **일부 지표가 좋아져도 다른 필수 지표를 악화시키는 변경은 배포하지 않았습니다.**

| 판단 | 결과 |
|---|---|
| Gov24 통합 뒤 출처 경쟁 보정 | 현재 평가셋에서 필요한 최소 보정만 적용 |
| cross-encoder 리랭커 | Gov24 일부 지표는 개선됐지만 Youth Recall@5·@10과 MRR이 악화되어 **No-Go** |
| 지역 검색 | 원본 지역 데이터 신뢰도가 부족해 public 경로에서 **비노출** |
| 현재 production | `RERANK=0`, 후보 30개, score cut·만료 제외·지역어 전처리 적용 |

리랭킹은 “얼마나 기여했는가”로 단정하지 않고, **어떤 지표를 개선하고 어떤 지표를 악화시키는지 분리해 측정**했습니다.
정확한 metric 표, 평가셋, 결과 JSON, 한계는 [검증 기록](docs/CUSTOM_SEARCH_MVP.md)과
[`eval/canonical_manifest.json`](eval/canonical_manifest.json)에 보존했습니다.

## 아키텍처

```text
[React / Vite]                 사용자 입력 (질문 + 나이)
      │  POST /api/ask
      ▼
[Spring Boot API]  ── 요청 검증 · 오케스트레이션 · Gemini 답변 생성
      │  POST /search
      ▼
[Python FastAPI · ML]  ── e5 질의 임베딩 → pgvector 검색(30) → production 보정/score cut (`RERANK=0`)
      │
      ▼
[Postgres + pgvector (Neon)]   정책 메타(구조화) + 본문 청크 벡터(768d)
```

## 기술 스택

| 영역 | 사용 기술 |
|---|---|
| 프론트 | React 18, Vite |
| API | Spring Boot 3.3 (Java 17), RestClient (Apache HttpClient5) |
| ML | Python, FastAPI, sentence-transformers |
| 임베딩 | `intfloat/multilingual-e5-base` (768d, 로컬 구동) |
| 리랭커 | `BAAI/bge-reranker-v2-m3` (평가·로컬 경로) |
| 생성 | Google Gemini `gemini-3.5-flash-lite` (Free Tier, `GEMINI_MODEL`로 교체 가능) |
| 저장소 | PostgreSQL + pgvector (Neon) |
| 인프라 | Cloud Run, GitHub Actions, GitHub Pages |
| 데이터 | 공공데이터포털 온통청년 청년정책 + 행정안전부 정부24 공공서비스(혜택) OpenAPI |

## 실행
`.env`에 `DATABASE_URL`(Neon), `YOUTH_API_KEY`, `DATA_GO_KR_KEY`, `GEMINI_API_KEY`가 필요합니다. `GEMINI_MODEL` 미설정 시 `gemini-3.5-flash-lite`가 사용됩니다.

```bash
# 1) 데이터 수집 + 임베딩 + 적재 (pgvector 지원 Postgres 필요)
cd ingest && python -m venv .venv && .venv\Scripts\activate
pip install -r requirements.txt -r ../ml-service/requirements.txt
python ingest_youth.py
python ingest_gov24.py --limit 5  # 먼저 공식 API 연결·필드 소량 확인
python ingest_gov24.py
python embed.py && python load_db.py

# 2) ML 서비스
cd ../ml-service && uvicorn app:app --port 8000

# 3) API (Spring Boot) — 다른 터미널
cd ../api && set GEMINI_API_KEY=... && gradlew bootRun

# 4) 프론트 — 다른 터미널
cd ../web && npm install && npm run dev   # http://localhost:5173
```

평가셋·측정 스크립트·결과는 `eval/`과 [검증 기록](docs/CUSTOM_SEARCH_MVP.md)에 있습니다.

## 만든 사람

**Jigwan Joe** — Backend · Data

- GitHub: [@jgjoe](https://github.com/jgjoe)
- Email: jigwan.joe@gmail.com

데이터 출처: 온통청년, 행정안전부 정부24 공공서비스(공공데이터포털)
