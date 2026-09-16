# 혜택나침반 (BenefitCompass)

**공식 정책을 여러 출처에서 질문 한 줄로 찾는 RAG 검색 서비스 — 검색 품질을 직접 만든 평가셋으로 측정합니다**

[![Live](https://img.shields.io/badge/live-demo-success)](https://jgjoe.github.io/benefit-compass)
[![prod-parity recall@1](https://img.shields.io/badge/prod--parity_recall%401-0.233-orange)](#검색-품질을-직접-측정했습니다)
[![Stack](https://img.shields.io/badge/stack-Spring%20Boot%20%2B%20FastAPI%20%2B%20pgvector-informational)](#아키텍처)
[![CI](https://img.shields.io/badge/CI-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)](.github/workflows)

정책과 혜택은 여러 공식 출처에 흩어져 있어 **정작 내가 받을 수 있는 게 뭔지 찾기가 어렵습니다.**
나이와 "월세 지원 받고 싶어" 같은 질문을 넣으면 관련 정책을 찾아 **근거와 함께** 답합니다.

데이터 수집·정제부터 임베딩·벡터검색·리랭킹·답변 생성, 배포와 운영 관측까지 혼자 만들었습니다.
기존 청년정책 검색은 **직접 만든 60문항 평가셋으로 쟀고**, 행정안전부 정부24 공공서비스 10,958건을 같은 경로에 합쳐 로컬 Neon 적재와 확장 검색 평가까지 완료했습니다.

> Custom Search 확장의 구현·평가 기록은 [검증 기록](docs/CUSTOM_SEARCH_MVP.md)에서 확인할 수 있습니다. 현재 공개 라이브 데모는 온통청년 + 정부24 통합 코퍼스를 사용하는 production 경로이며, 실제 public rollout 및 운영 topology는 [Public Rollout 기록](docs/P3_PUBLIC_ROLLOUT.md)에 기록되어 있습니다.

**[라이브 데모](https://jgjoe.github.io/benefit-compass)** — Cloud Run scale-to-zero 구성이라 첫 요청은 인스턴스와 모델을 올리는 시간이 걸립니다.

![월세 지원 질문의 실제 검색 결과](docs/images/search-result.png)

---

## 검색 품질을 직접 측정했습니다

RAG는 "그럴듯한 답"이 나오면 잘 되는 것처럼 보입니다. 그래서 **질문 60개에 정답 정책을 라벨링한 평가셋**을
만들고, 정답이 실제로 상위에 오는지 수치로 확인했습니다.

### 후보 랭킹 보정

Gov24 10,958건을 추가한 뒤 기존 60문항의 후보 검색 Recall@1은 `0.3167`로 하락했다. 질의에
`청년`·`대학생`·`사회초년생`이 명시되고 알려진 Gov24 기관명이 없을 때만 `youth` 출처의 거리에
`0.015`를 보정했다.

| 평가셋·지표 | 확장 후 무보정 | 최소 보정 후 |
|---|---:|---:|
| 기존 60문항 Recall@1 | 0.3167 | **0.3333** |
| 기존 60문항 Recall@5 | 0.6667 | 0.6667 |
| 기존 60문항 Recall@10 | 0.7333 | **0.7833** |
| 기존 60문항 MRR@10 | 0.4560 | **0.4693** |
| 신규 Gov24 21문항 Recall@1 | 0.2857 | 0.2857 |
| 신규 Gov24 21문항 Recall@5 | 0.4762 | 0.4762 |
| 신규 Gov24 21문항 Recall@10 | 0.7143 | 0.7143 |
| 신규 Gov24 21문항 MRR@10 | 0.3901 | 0.3901 |

이 표는 source competition만 분리한 후보 랭킹 진단이다. production의 만료 정책 제외,
지역어 전처리와 score cut을 적용한 배포 정확도로 해석하지 않는다.

### Production-parity 리랭커 평가: No-Go

실제 `/search`와 같은 SQL·질의 전처리·후보 30개·score cut을 공유해 `RERANK=0`과
`bge-reranker-v2-m3`를 다시 비교했다.

| 평가셋·지표 | `RERANK=0` | `RERANK=1` |
|---|---:|---:|
| 기존 60문항 Recall@1 | 0.2000 | **0.2500** |
| 기존 60문항 Recall@5 | **0.4000** | 0.3333 |
| 기존 60문항 Recall@10 | **0.4667** | 0.3333 |
| 기존 60문항 MRR@10 | **0.2881** | 0.2817 |
| 신규 Gov24 21문항 Recall@1 | 0.2857 | 0.2857 |
| 신규 Gov24 21문항 Recall@5 | 0.4762 | **0.6190** |
| 신규 Gov24 21문항 Recall@10 | 0.6190 | 0.6190 |
| 신규 Gov24 21문항 MRR@10 | 0.3798 | **0.4222** |
리랭커는 Gov24 21문항 일부 지표를 높였지만 기존 youth 60문항의 Recall@5·@10과 MRR을 악화시켰다.
따라서 전체 검색에는 채택하지 않았고 배포 구성은 `RERANK=0`을 유지한다. `0.015`도 현재 평가에서
선택한 최소값일 뿐 일반화된 production 최적값으로 간주하지 않는다. 평가셋·결과 JSON·한계는
[검증 기록](docs/CUSTOM_SEARCH_MVP.md)에 남겼다.

### Canonical production-parity baseline — P0 동결 (2026-08-29)

현재 production과 동일한 검색 계약(`RERANK=0`, `CANDIDATES=30`, `COSINE_MIN=0.78`, `LEXICAL 0.01`, `strip_region`, 만료 제외)으로 재현한 **현재 기준선**이다. lexical `0 → 0.01` 비교는 `eval/run_eval.py --lexical-bias`로 같은 계약에서 재현했다.

| 평가셋·지표 | lexical `0` | lexical `0.01` (production) |
|---|---:|---:|
| 기존 60문항 Recall@1 | 0.2000 | **0.2333** |
| 기존 60문항 Recall@5 | 0.4000 | **0.4667** |
| 기존 60문항 Recall@10 | 0.4667 | **0.5167** |
| 기존 60문항 MRR@10 | 0.2881 | **0.3281** |
| 신규 Gov24 21문항 Recall@1 | 0.2857 | 0.2857 |
| 신규 Gov24 21문항 Recall@5 | 0.4762 | **0.7143** |
| 신규 Gov24 21문항 Recall@10 | 0.6190 | **0.7619** |
| 신규 Gov24 21문항 MRR@10 | 0.3798 | **0.4222** |

위 수치는 `eval/canonical_youth_production_parity.json`, `eval/canonical_gov24_production_parity.json`에서 재현된다. historical 실험 파일(`results_after_*`, `results_expansion_*`)은 보존했고, 현재 기준선은 `eval/canonical_manifest.json`에서 한 번에 추적한다. `Recall@1 0.40 → 0.52` 같은 과거 수치는 만료/지역어/score cut 없는 후보 진단이므로 production 정확도로 해석하지 않는다.

평가셋 생성(`eval/make_evalset.py`)과 측정(`eval/run_eval.py`, `eval/run_eval_rerank.py`) 스크립트,
평가셋 원본과 측정 결과 JSON을 저장소에 공개했다. 같은 명령으로 다시 잴 수 있다.

**적재 규모**: 청년정책 **2,631건**을 정제해 **3,083개 청크**로 적재했고, 임베딩 누락은 **0건**입니다.

## 왜 이렇게 만들었나

### RAG를 한 덩어리로 두지 않았다

검색 결과를 LLM에 통째로 넘기면 품질이 나빠졌을 때 **어디가 문제인지 알 수 없습니다.**
임베딩 · 벡터검색 · 선택적 리랭킹 · 생성을 단계로 쪼개 두니 단계별로 따로 측정할 수 있었고,
위 표처럼 "리랭킹이 1순위 정답률에 얼마를 기여하는지"를 분리해 말할 수 있게 됐습니다.

### 답변은 검색된 정책만 근거로 쓴다

모델이 정책을 지어내면 사용자가 존재하지 않는 지원금을 신청하러 갑니다.
검색 결과 범위 안에서만 답하게 하고 정책명을 함께 인용하게 했습니다.
마땅한 결과가 없으면 "없다"고 단정하지 않고 다시 검색해 보도록 안내합니다.

### 지역 필터는 데이터를 믿을 수 없어 노출을 끊었다

`zipCd`를 법정동코드로 적재해 검색 SQL에 지역 조건까지 걸었는데, **서울로 필터링해도 함안군 정책이 통과**했습니다.
진단 스크립트(`ingest/inspect_region.py`)로 원본을 덤프해 보니 지자체 정책에 타지역 코드가 섞여 있었고
기관명도 부서명뿐인 경우가 많았습니다. 기관명 기반 보강 필터를 덧대 봤지만 **원본이 틀린 이상 신뢰할 수 없다고 판단**해
사용자 노출에서 제외했습니다.

**기능을 지우지 않고 노출만 끊었습니다.** 웹 UI에서 지역 입력을 제거해 사용자 경로에서는 지역 검색이 사라졌지만,
`ml-service/app.py`의 `region_filter`와 검색 SQL의 지역 조건은 코드에 그대로 남겨 두었습니다.
원본 데이터를 정제한 뒤 `ingest/search.py --region`으로 **바로 다시 검증하기 위해서**입니다.

다만 **HTTP API는 `region`을 받지 않습니다.** `POST /api/policies/recommend`·`POST /api/ask`에
`region`이 들어오면 빈 문자열이라도 `400 INVALID_REQUEST`로 거절합니다.
신뢰할 수 없다고 판단한 필터가 조용히 무시된 채 통과한 것처럼 보이는 편이 더 위험하기 때문입니다.
지역 검증은 CLI(`ingest/search.py --region`)로만 합니다.
질의에 섞인 지역어는 검색 잡음이 되므로 `strip_region`으로 제거합니다.

### 임베딩은 외부 API 대신 로컬 모델

처음엔 Gemini 임베딩을 쓰려 했으나 무료 티어가 분당·일일 한도에 금방 걸렸습니다.
한국어에 강한 `multilingual-e5-base`를 컨테이너에서 직접 돌려 **모델 API 호출 한도를 제거**했습니다.
(실행 인프라 비용은 별도입니다.)

### API는 Spring Boot, ML은 Python

ML 라이브러리는 Python 생태계가 편하고 비즈니스 로직은 Spring Boot가 낫습니다.
둘을 한 프로세스에 두지 않고 서비스로 나눴습니다.

## 아키텍처

```text
[React / Vite]                 사용자 입력 (질문 + 나이)
      │  POST /api/ask
      ▼
[Spring Boot API]  ── 요청 검증 · 오케스트레이션 · Gemini 답변 생성
      │  POST /search
      ▼
[Python FastAPI · ML]  ── e5 질의 임베딩 → pgvector 검색(30) → 선택적 bge 리랭킹 → 임계값 컷
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

## 운영과 관측

배포하고 끝내지 않고 **볼 수 있게** 만들었습니다.

- `/actuator/prometheus` — endpoint·상태 구간별 지연시간, 검색 결과/무결과 수집
- `/actuator/health` — 배포 상태 확인
- 모든 API 응답에 `X-Request-ID`를 넣어 장애 로그를 추적
- **질문 원문과 나이는 로그·메트릭에 저장하지 않습니다** (장애 조사 목적으로도 남기지 않음)

### 콜드스타트를 구간으로 분해했다

무료 티어 scale-to-zero 구성이라 첫 요청이 느립니다. **느리다는 체감을 수치로 바꾸는 것부터** 했습니다.

1. 요청 ID와 구간 header를 넣어 콜드 경로를 **API↔ML / 모델 준비 / 임베딩 / DB 연결·쿼리 / 생성**으로 분해
2. 공개 traffic을 바꾸지 않은 **0% revision**에서, 15분 유휴 뒤 before/after를 동시 호출하는 절차로 **5쌍 반복 측정**
3. 지배 구간은 `api_ml_transport` 중앙값 **26.9초** — ML scale-from-zero + 모델 준비 대기 + 큐. 모델 로딩만 23~24초
4. 비용이 들지 않는 레버만 적용: `/ready` startup probe로 모델 준비 전 트래픽 차단, 런타임 모델 허브 의존 제거

결과적으로 **모델 로딩 중앙값은 24.2초 → 23.1초(-4.73%)로 줄었지만 end-to-end 개선은 확정하지 못했습니다**
(중앙값 +0.27%, pair별 편차가 큼). 검증된 개선은 런타임 Hub 의존 제거와 정상 동작이며, **사용자 지연 개선은 주장하지 않습니다.**
남은 레버인 최소 인스턴스 상시 기동은 효과가 확실하지만 상시 과금이라 개인 프로젝트에서는 적용하지 않았습니다.

> 원본 CSV·revision·한계: [Production Lab 2](docs/operations/PRODUCTION_LAB_2_2026-07-21.md) ·
> [운영 기준선](docs/operations/BASELINE_2026-07-14.md) · [SLO 초안](docs/operations/SLO.md) · [런북](docs/operations/RUNBOOK.md)

## 실행 방법
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

평가 재현 (canonical — 현재 production 계약):

```bash
python eval/run_data_quality.py
# Youth 60 — production parity (lexical 0.01) 및 lexical ablation 비교
python eval/run_eval.py --eval-file eval/evalset.jsonl --output eval/canonical_youth_production_parity.json --lexical-bias 0.01
python eval/run_eval.py --eval-file eval/evalset.jsonl --output eval/canonical_youth_production_lexical_0.json --lexical-bias 0
# Gov24 21 — 동일 계약
python eval/run_eval.py --eval-file eval/expansion_evalset.jsonl --output eval/canonical_gov24_production_parity.json --lexical-bias 0.01
python eval/run_eval.py --eval-file eval/expansion_evalset.jsonl --output eval/canonical_gov24_production_lexical_0.json --lexical-bias 0
# 36-case hard-negative 진단 (retrieval-level, Gemini 없이)
python eval/run_hard_negative_eval.py --eval-file eval/expansion_api_evalset.jsonl --output eval/canonical_hard_negative_36_production_parity.json --lexical-bias 0.01
```

historical 실험 파일(`eval/results_before_expansion.json`, `eval/results_after_*` 등)은 보존했다. 상세 계약과 결과 해석은 [검증 기록](docs/CUSTOM_SEARCH_MVP.md)과 `eval/canonical_manifest.json`을 따른다.

저장소에 커밋된 canonical artifact는 clean evaluator commit `58dff80`에서 저장소 밖 임시 디렉터리로 생성해 `git_dirty=false`를 확인한 뒤 `eval/`에 복사했다. 아래 명령을 tracked canonical 경로에 직접 순차 실행하면 지표는 재현되지만, 첫 출력으로 working tree가 변경된 뒤 실행되는 artifact에는 `git_dirty=true`가 기록될 수 있다.

## 측정 조건과 범위

- 평가 수치는 직접 라벨링한 기존 60문항과 Gov24 21문항 기준입니다. 표본이 작아 1문항 변화의 유의성을 판단하지 않았습니다. canonical 결과는 `eval/canonical_youth_production_parity.json` 등에서 `generated_at`·`git_commit`·`corpus`와 함께 재현된다.
- 공개 경로는 무료 인스턴스의 CPU·메모리 조건에 맞춰 **리랭킹을 끈 구성(`RERANK=0`)으로 배포**했습니다. production-parity 평가에서도 전체 채택 기준을 충족하지 못해 이 구성을 유지한다. canonical baseline은 `RERANK=0`, `CANDIDATES=30`, `COSINE_MIN=0.78`, `LEXICAL 0.01`이다.
- **지역 검색은 제공하지 않습니다.** 원본 지역코드 품질 문제로 노출을 끊은 상태이며, 데이터 정제나 신뢰할 수 있는 출처 확보가 선행 과제입니다.
- **코드와 현재 공개 경로는 온통청년 + 정부24 복수 출처를 지원합니다.** production DB에는 정책 13,589건과 청크 17,609건이 있으며, P3 rollout에서 public API의 youth/Gov24 검색 경로를 검증했습니다. 다만 Gov24 21문항 평가는 작은 표본이므로 전반적인 검색 품질 향상을 일반화하지 않습니다. 현재 public rollout 근거는 [Public Rollout 기록](docs/P3_PUBLIC_ROLLOUT.md)을 따릅니다.
- SLO 문서의 목표값은 **목표이며 달성 성과가 아닙니다.**
## 만든 사람

**Jigwan Joe** — Backend · Data

- GitHub: [@jgjoe](https://github.com/jgjoe)
- Email: jigwan.joe@gmail.com

비영리 학습·포트폴리오 프로젝트입니다. 데이터 출처는 온통청년과 행정안전부 정부24 공공서비스(공공데이터포털)입니다.
