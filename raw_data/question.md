# 데이터 분석가 채용 과제 안내

## 0. 과제 개요

저희는 **리워드 기반 건강 습관 앱 서비스**를 운영하고 있습니다.

유저는 앱을 설치하고 가입한 뒤, 매일 걷기/앱 사용/광고 시청 등을 통해 포인트를 적립할 수 있습니다.

이번 과제의 목적은, 지원자님이

1. **데이터 인프라 / 분석 시스템을 어떻게 설계하는지**
2. **실험(A/B 테스트)을 어떻게 설계하고, 실제 데이터를 분석해 의사결정까지 연결하는지**

를 실전과 유사한 환경에서 확인해보는 것입니다.

> ⚠️ 유의사항
> 
> - 이 데이터는 실제 서비스 데이터가 아닌, **시뮬레이션된 가상 데이터**입니다.
> - 비즈니스 정합성 100%가 아니라, **분석 로직과 사고과정을 보는 목적**입니다.
> - 정답/오답보다는 “어떻게 생각하고 풀어가는지”를 더 중요하게 봅니다.

---

## 1. 제공 데이터셋

아래 5개의 CSV를 제공합니다. (지원자에게는 실제 다운로드 링크 또는 파일 첨부)

[users_large.csv](attachment:d72b9871-ab38-4ffb-9af1-fbb124ff4dca:users_large.csv)

[raw_app_events_large.csv](attachment:8bded625-74e5-468d-a6ca-15d62bd5cb30:raw_app_events_large.csv)

[raw_points_large.csv](attachment:677685bc-5eff-4235-926b-5001ae192f05:raw_points_large.csv)

[raw_ads_revenue_large.csv](attachment:07213cc3-55c4-4a95-b686-f6836f3eeb8e:raw_ads_revenue_large.csv)

[ab_experiment_user_metrics_large.csv](attachment:62558cdc-36ac-4501-84da-1de817f50054:ab_experiment_user_metrics_large.csv)

### 1-1. users_large.csv

**설명**: 가입 유저 마스터 정보

| 컬럼명 | 타입 | 설명 |
| --- | --- | --- |
| user_id | STRING | 유저 고유 ID |
| signup_at | DATETIME | 가입 시각 (`YYYY-MM-DD HH:MM:SS`) |
| country | STRING | 국가 코드 (KR/JP/US) |
| marketing_channel | STRING | 유입 채널 |
| device_os | STRING | OS 정보 (Android / iOS) |

---

### 1-2. raw_app_events_large.csv

**설명**: 앱 내 사용자 행동 로그

| 컬럼명 | 타입 | 설명 |
| --- | --- | --- |
| event_time | DATETIME | 이벤트 발생 시각 |
| user_id | STRING | 유저 ID |
| event_name | STRING | 이벤트명 (`app_open`, `step_update`, `ad_impression`, `ad_click`, `reward_claim`) |
| platform | STRING | 플랫폼 (`android`, `ios`) |
| country | STRING | 국가 코드 |
| device_id | STRING | 디바이스 ID |
| event_properties | STRING | 추가 속성 (`step_delta=...`, `placement=...`, `reason=...` 등) |

> 기간: 2025-01-01 ~ 2025-01-07 (7일)
> 

---

### 1-3. raw_points_large.csv

**설명**: 포인트 적립/차감 이력

| 컬럼명 | 타입 | 설명 |
| --- | --- | --- |
| point_id | STRING | 포인트 트랜잭션 ID |
| created_at | DATETIME | 발생 시각 |
| user_id | STRING | 유저 ID |
| point_delta | INTEGER | 포인트 증감 (적립: 양수, 차감: 음수) |
| reason | STRING | 사유 (`steps_reward`, `ad_reward` 등) |
| campaign_id | STRING | 캠페인 ID |

---

### 1-4. raw_ads_revenue_large.csv

**설명**: 광고 네트워크 리포트 (일자 × 국가 × 플랫폼 × 네트워크 집계)

| 컬럼명 | 타입 | 설명 |
| --- | --- | --- |
| date | DATE | 기준 일자 |
| country | STRING | 국가 코드 |
| platform | STRING | 플랫폼 (`android`, `ios`) |
| ad_network | STRING | 광고 네트워크명 |
| impressions | INTEGER | 노출 수 |
| clicks | INTEGER | 클릭 수 |
| revenue | FLOAT | 광고 수익 (USD) |

---

### 1-5. ab_experiment_user_metrics_large.csv

**설명**: 리워드 정책 A/B 테스트 결과(유저 단위 집계)

| 컬럼명 | 타입 | 설명 |
| --- | --- | --- |
| user_id | STRING | 유저 ID |
| group | STRING | 실험 그룹 (`control`, `test`) |
| signup_date | DATE | 가입 일자 |
| country | STRING | 국가 코드 |
| baseline_steps_7d | INTEGER | 실험 전 7일 동안의 기준 걸음 수 (정의는 과제에서 명시해 주세요) |
| steps_7d | INTEGER | 실험 기간 7일 동안 총 걸음 수 |
| reward_points_7d | INTEGER | 실험 기간 7일 동안 지급된 포인트 |
| ad_revenue_7d | FLOAT | 실험 기간 동안 유저 기준 광고 수익(USD) |
| is_retained_d7 | INTEGER | D7 리텐션 여부 (1=잔존, 0=이탈) |

---

## 2. 과제 전체 안내

- **권장 소요 시간**: 2시간 내외
- **제출 형식**
    - 분석/설계 문서: Notion / Google Doc / PDF 등 자유
    - SQL / Python 코드: GitHub 링크 또는 압축 파일
    - (선택) 간단한 대시보드/시각화: 스크린샷 또는 링크

> ⚠️ “완벽한 정답”보다, 문제 정의 → 설계 → 구현/분석 → 인사이트로 이어지는 사고 구조와 커뮤니케이션을 중요하게 봅니다.
> 

---

## 3. Part 1 – 데이터 인프라 / 시스템 설계

### 3-1. 목표

아래 요구 사항을 만족하는 **데이터 인프라 및 마트 구조**를 설계하고,

핵심 지표를 계산할 수 있는 **ETL/쿼리 설계 & 데이터 품질 관리 방안**을 제시해 주세요.

### 3-2. 수행 과제

### (1) 전사 데이터 레이어 & 테이블 설계

아래 RAW 데이터를 기반으로, Data Lake / Warehouse / Mart 혹은 Medallion 구조(Silver/Gold 등)를 설계해 주세요.

- `users_large`
- `raw_app_events_large`
- `raw_points_large`
- `raw_ads_revenue_large`

**포함해야 할 내용**

1. **데이터 레이어 구조**
    - 각 레이어의 역할과 목적 설명
2. **핵심 테이블/뷰 설계**
    - 각 테이블에 대해:
        - Grain / PK
        - 주요 컬럼 목록 및 정의
        - 어떤 RAW 데이터에서 어떤 로직으로 집계되는지 간단히 설명
3. **엔드투엔드 데이터 플로우**
    - 박스/화살표 다이어그램 또는 텍스트 형태로,
    - “원천 데이터 → 스테이징 → 마트 → 대시보드/분석” 흐름을 설명

---

### (2) DAU / 리텐션 / 리워드 / 광고 지표 파이프라인 설계

다음 지표들이 **매일 안정적으로 산출**될 수 있는 ETL 로직/쿼리를 설계해 주세요.

- 일별 DAU (Day Active Users)
- 간단한 코호트 리텐션 (예: Day1 / Day7)
- 유저별 일별 걸음 수, 포인트 적립, 광고 시청 수
- 일별 광고 수익 (country, platform 단위까지 집계)

**포함해야 할 내용**

1. **DAU / 리텐션 정의 및 SQL 또는 PySpark 쿼리 예시**
    - DAU 정의 (어떤 이벤트를 기준으로 할지)
    - D1 / D7 리텐션 정의
    - 이를 계산하기 위한 쿼리 또는 의사 코드
2. **유저-일 마트 생성 로직**
3. **데이터 품질(DQ) 룰 & 모니터링**
    - ETL 과정에서 체크해야 할 DQ rule 5개 이상
    - Airflow/Batch Orchestration을 사용한다고 가정하고,
        - DAG 구조 (의존 관계, 실패 시 재처리 전략)를 간단히 설명

---

## 4. Part 2 – 실험 설계 & A/B 테스트 분석

### 4-1. 실험 시나리오

**리워드 정책 강화 실험**

- 기존: 5,000보 달성 시 10포인트 지급
- 실험군(Test): 첫 7일 동안 5,000보 달성 시 더 높은 포인트(예: 15포인트)를 지급하는 정책

`ab_experiment_user_metrics_large.csv`는 이 실험이 **일정 기간 진행된 뒤, 유저 단위로 7일 데이터를 집계한 결과**라고 가정합니다.

---

### 4-2. 실험 설계

아래 내용에 대해서 생각을 정리해주세요.

1. **실험 목표 & 가설**
    - Primary Hypothesis 예시:
        - “강화된 리워드 정책(Test)은 Control 대비 7일 동안의 총 걸음 수를 **X% 이상 증가**시킨다.”
    - Secondary / Guardrail 지표 설정
        - 예: 광고 수익, 리텐션, 포인트 인플레이션 등
2. **실험 설계 시 고려 사항**
    - 랜덤 배정 방식
    - 샘플 사이즈 / 기간에 영향을 주는 요소
    - 유의수준(α), 검정력(1-β)에 대한 생각
3. **설계 상의 리스크/함정 예시**
    - 예: 그룹간 baseline 차이, 특정 국가에만 실험이 편향되는 경우 등

---

### 4-3. A/B 테스트 결과 분석

`ab_experiment_user_metrics_large.csv`를 활용해 다음을 분석해 주세요.

1. **그룹별( control vs test ) 주요 지표 비교**
    - 평균 `steps_7d`
    - 평균 `reward_points_7d`
    - 평균 `ad_revenue_7d`
    - D7 리텐션율 (`is_retained_d7`)
2. **통계적 유의성 검정**
    - 어떤 검정 방법을 선택했고, 어떤 가정을 두었는지 간단하게 설명
3. **실험 결과에 대한 분석**
    - “실험이 성공적이라고 판단하는지 / 아닌지”
    - “어떤 후속 액션을 추천하는지” (전량 배포, 일부 세그먼트만, 추가 실험 필요 등)
    - 텍스트 혹은 슬라이드 캡처 형태 모두 가능

---