# 리워드 광고 성과 분석 및 광고 점수화 모델링 프로젝트

## 프로젝트 개요

리워드 광고의 한 달치 운영 데이터를 기반으로 광고 성과를 다각도로 분석하고, 광고 운영 의사결정을 지원하기 위한 **광고 점수화 모델**과 **조기부진 예측 모델**을 구축한 프로젝트입니다.

기존 광고 평가는 클릭 수나 완료율(CVR)처럼 단일 지표 중심으로 이루어지기 쉽습니다. 하지만 리워드 광고는 클릭이 많아도 완료로 이어지지 않거나, 완료가 많아도 리워드 비용이 커서 마진이 낮을 수 있습니다. 또한 특정 광고가 초반에만 반응이 좋고 이후 급격히 감소한다면 안정적인 운영 대상으로 보기 어렵습니다.

따라서 본 프로젝트에서는 광고를 다음 네 가지 축으로 평가했습니다.

- **반응성**: 유저가 광고에 얼마나 잘 반응하고, 클릭이 완료로 이어지는가
- **수익성**: 광고 성과가 실제 마진으로 연결되는가
- **안정성**: 성과가 일시적이지 않고 관측 기간 동안 유지되는가
- **규모**: 광고가 전체 운영 성과에 미치는 영향력이 어느 정도인가

또한 기존 광고 유형(`ads_type`, `ads_category`)이 일관된 기준으로 분류되지 않는 문제를 발견하고, 광고 문구 기반의 **BERTopic + Rule-based Cascade 분류**를 적용하여 광고의 매체(`final_media`)와 행동유형(`final_action`)을 재분류했습니다.

---

## 프로젝트 목표

### 1. 리워드 광고 성과 진단

- 클릭 → 완료 → 재참여 → 수익성으로 이어지는 광고 퍼널 구조 파악
- 광고 유형, 보상금액, 시간대, 요일, CTIT 등에 따른 반응성 차이 분석
- 광고별 마진, 클릭당 마진, 완료당 마진을 활용한 수익성 분석
- 광고 성과가 지속적으로 유지되는지 안정성 분석
- 일부 광고에 성과가 과도하게 집중되는지 규모 및 집중도 분석

### 2. 광고 유형 재분류

- 기존 광고 유형과 카테고리의 분류 기준 불일치 문제 확인
- 광고명, 광고 설명, 저장 방식, URL, 앱 패키지 등의 텍스트 정보를 활용
- BERTopic을 최종 분류기로 사용하지 않고, 검수 보조 및 unknown 보완 용도로 활용
- Rule-based Cascade 구조를 통해 희소 라벨이 다수 라벨에 흡수되는 문제 완화
- 최종적으로 `final_media`, `final_action` 기반의 새로운 분석 축 생성

### 3. 광고 점수화 모델 개발

- 최종 성과를 기반으로 광고 품질 점수(`quality_score`) 산출
- 품질 점수 상위 25% 광고를 고성과 광고로 정의
- 광고 등록 시점에 알 수 있는 속성 정보만으로 고성과 광고를 사전 예측
- 예측 확률 기반 S/A/B/C/D 등급화

### 4. 조기부진 예측 모델 개발

- 광고 시작 후 D+3 시점의 초기 실적을 활용
- 최종 품질 점수 하위 25% 광고를 조기에 탐지
- 완료 수 계열 누수 피처를 제거하고 운영자 우선순위화 보조 모델로 설계
- 자동 중단이 아니라 점검 우선순위 선정 목적으로 활용

---

## 분석 및 모델링 흐름

```text
01_01_ad_classification_bertopic.ipynb
    └─ 광고 문구 기반 BERTopic + Rule-based 재분류

01_02_preprocessing.ipynb
    └─ 원천 테이블 전처리 및 분석용 parquet 생성

02_EDA.ipynb
    └─ 반응성 / 수익성 / 안정성 / 규모 기반 EDA

03_00_ML_EDA.ipynb
    └─ 모델링 전 데이터 및 피처 진단

03_01_ML_mart.ipynb
    └─ 모델 학습용 데이터마트 생성 및 누수 방지 설계

03_02_ML1.ipynb
    └─ Model 1: 광고 고성과 예측 + 점수화 + 등급화

03_03_ML2.ipynb
    └─ Model 2: 광고 조기부진 예측

model1_best_params.json
    └─ Model 1 최종 XGBoost 하이퍼파라미터

model2_best_params.json
    └─ Model 2 최종 RandomForest 하이퍼파라미터
```

---

## 데이터 구성

본 프로젝트는 리워드 광고의 클릭, 완료, 재무, 광고 속성, 광고 운영 캘린더, 유저 활동 데이터를 결합하여 분석했습니다.

### 주요 테이블

| 테이블 | 설명 |
|---|---|
| `main_funnel.parquet` | 클릭 → 완료 퍼널 기준 테이블 |
| `ads_join_info_labeled.parquet` | 광고 참여 로그 및 완료 여부 라벨 |
| `finance_clean1.parquet`, `finance_clean2.parquet` | 광고별 비용, 수익, 마진 정보 |
| `ad_outcome.parquet` | 광고 성과 요약 |
| `ad_attr_map.parquet` | 광고 속성 매핑 |
| `ad_master_clean.parquet` | 광고 마스터 정보 |
| `sched_clean.parquet` | 광고 운영 캘린더 |
| `user_daily_activity_clean.parquet` | 유저 일자별 활동 |
| `ive_ad_classification.parquet` | BERTopic 및 Rule-based 재분류 결과 |

> 원본 데이터 및 parquet 산출물은 용량과 내부 데이터 이슈로 GitHub에는 포함하지 않습니다.  
> GitHub에는 분석 코드, README, requirements, 핵심 시각화 이미지만 업로드하는 것을 권장합니다.

---

## 기술 스택

### Language

- Python
- SQL

### Data Processing

- Pandas
- Polars
- NumPy

### Visualization

- Matplotlib
- Seaborn

### Machine Learning

- Scikit-learn
- XGBoost
- LightGBM
- RandomForest
- Logistic Regression
- Optuna
- SHAP

### NLP / Topic Modeling

- SentenceTransformer
- BERTopic
- UMAP
- HDBSCAN
- TF-IDF

### Storage

- Parquet
- JSON
- Joblib

---

# 1. 광고 유형 재분류: BERTopic + Rule-based Cascade

## 문제 인식

EDA 과정에서 기존 광고 유형(`ads_type`)과 광고 카테고리(`ads_category`)가 일관된 기준으로 분류되어 있지 않다는 문제가 확인되었습니다.

예를 들어, 기존 광고 유형은 매체 기준과 행동 기준이 섞여 있었습니다.

- `네이버`, `인스타`, `유튜브` → 매체 기준
- `설치형`, `실행형`, `클릭형`, `참여형` → 행동 기준
- `CPS(물건구매)` → 과금 또는 구매 행동 기준

이처럼 기준이 혼재되어 있으면 광고 유형별 성과 차이를 해석할 때, 실제로는 매체 효과인지 행동 난이도 효과인지 분리하기 어렵습니다.

## 재분류 방향

따라서 광고를 두 축으로 분리했습니다.

| 축 | 설명 | 예시 |
|---|---|---|
| `final_media` | 광고가 연결되는 매체/플랫폼 | 네이버, 유튜브, 인스타그램, 앱, 웹 등 |
| `final_action` | 유저가 수행해야 하는 행동 | 클릭, 설치, 실행, 가입, 구매, 시청, 참여 등 |

## 분류 방식

### 1. URL / 도메인 / 앱 패키지 추출

일반적인 텍스트 전처리에서는 URL을 제거하는 경우가 많지만, 본 프로젝트에서는 URL을 중요한 단서로 활용했습니다.

예시:

- `youtube.com` → 유튜브 관련 광고 가능성
- `instagram.com` → 인스타그램 관련 광고 가능성
- `play.google.com` → 앱 설치/실행 관련 광고 가능성
- 앱 패키지명 → 앱 광고 여부 판단

### 2. 강한 텍스트와 약한 텍스트 분리

광고 분류에 사용되는 텍스트를 신뢰도에 따라 구분했습니다.

| 구분 | 설명 |
|---|---|
| strong text | 광고명, 광고 설명, 참여 방법 등 직접적인 미션 단서 |
| weak text | 이미지 URL, 보조 링크 등 오탐 가능성이 있는 보조 단서 |
| classification doc | strong text + 도메인 + 앱 패키지를 결합한 최종 분류 문서 |

### 3. Rule-based Cascade 분류

기존 방식처럼 모든 라벨을 동시에 점수화하고 argmax로 고르는 방식은 다수 라벨이 유리합니다.  
이를 보완하기 위해 희소 라벨을 먼저 탐지하고, 남은 광고를 다음 단계로 넘기는 cascade 구조를 적용했습니다.

#### Media 분류 순서

| 단계 | 기준 |
|---|---|
| 1 | SNS 직접 단서 |
| 2 | 도메인 매칭 |
| 3 | 앱 패키지 / 앱 명시 |
| 4 | 웹 단서 |
| 5 | 네이버 직접 단서 |
| 6 | 네이버 정황 단서 |
| 7 | 기존 ads_type 기반 보강 |
| 8 | unknown |

#### Action 분류 순서

| 단계 | 기준 |
|---|---|
| 1 | 구매 / 가입 / 설치 / 실행 / 시청 / 노출 / 클릭 키워드 |
| 2 | 기존 ads_type 기반 강제 매핑 |
| 3 | 참여형 매핑 |
| 4 | unknown |

### 4. BERTopic 활용

BERTopic은 최종 정답 분류기로 사용하지 않았습니다.  
대신 다음 목적의 보조 도구로 활용했습니다.

- rule 기반으로 분류되지 않은 unknown 광고의 후보 라벨 제안
- 비슷한 광고 문구끼리 topic으로 묶어 검수 후보 축소
- topic별 대표 문서 확인
- topic 내부 rule 라벨 비율을 활용한 신뢰도 판단

### 5. 주요 개선사항

| 기존 문제 | 개선 방향 |
|---|---|
| 설치형 광고가 잘못 분류됨 | `ads_type` 강제 매핑을 강한 신호로 사용 |
| action_click이 과도하게 많음 | 근거 없는 클릭 기본값 제거, unknown 유지 |
| media_unknown이 과도하게 많음 | save_way 다수결 후처리 적용 |
| 라벨 간 오염 발생 | negative keywords 도입 |
| apply(axis=1)로 메모리 비효율 | numpy vectorized 방식으로 개선 |
| 토픽 유사도 margin이 낮음 | threshold를 보수적으로 상향 |

---

# 2. 전처리

## 전처리 원칙

전처리에서는 날짜 기준을 명확히 구분했습니다.

| 날짜 컬럼 | 사용 목적 |
|---|---|
| `click_date` | 클릭 수, DAU, 재방문, 행동 관찰 지표 |
| `regdate` | 완료 수, 적립 처리 시점, 완료 기준 성과, 마진 발생 시점 |

클릭과 완료는 발생 시점이 다를 수 있기 때문에, 각 지표의 기준 날짜를 분리해 해석했습니다.

## 주요 전처리 내용

### 재무 테이블

- 클릭일 / 완료일 파생 컬럼 생성
- 요일 파생 컬럼 생성
- `ive_margin = adv_cost - earn_cost` 명시 생성
- 특수 마진 그룹 플래그 생성
- 마진 분석용 정상 데이터 분리
- 음수 마진 및 재무 이상치 검증

### 광고 참여 및 적립 테이블

- 클릭 로그와 완료 로그 연결
- 클릭 후 완료 여부 생성
- CTIT(Click To Install/Complete Time) 계산
- 동일일 완료 여부 및 지연 완료 여부 생성
- 광고별 클릭 수, 완료 수, CVR 집계

### 광고 속성 테이블

- 광고 유형 라벨 매핑
- 광고 카테고리 정리
- 보상금액 구간(`reward_band`) 생성
- 광고 시작일/종료일 및 운영일차 계산

### 분석 테이블

- `main_funnel`
- `fact_click_reward`
- `ad_outcome`
- `ads_join_info_labeled`
- `ad_mart_v5`

---

# 3. EDA: 리워드 광고 평가 축 설계

## 전체 퍼널 진단

한 달치 리워드 광고 데이터의 전체 퍼널은 다음과 같이 요약되었습니다.

| 지표 | 값 |
|---|---:|
| 클릭 수 | 3,181,777 |
| 완료 수 | 1,383,784 |
| 완료율(CVR) | 43.49% |

이후 분석은 단순 클릭 수가 아니라, 클릭이 완료와 재참여, 마진으로 이어지는지를 중심으로 진행했습니다.

---

## 3-1. 반응성 분석

반응성은 유저가 광고에 얼마나 잘 반응하고, 클릭이 완료로 이어지는지를 보는 축입니다.

### 주요 분석 항목

- 광고유형별 클릭 수 / 완료 수 / 집계 CVR
- 광고 단위 CVR 분포
- 광고 카테고리별 성과
- 보상금액 구간별 클릭 수, 완료 수, CVR
- CTIT와 CVR 관계
- 시간대별 클릭 수, 완료율, 평균 CTIT
- 요일 × 시간대 반응성
- BERTopic 행동유형 기준 시간대 반응성

### 핵심 해석

- 클릭 수가 많은 유형이 반드시 CVR도 높은 것은 아니었습니다.
- 광고 유형별 CVR 차이가 존재하므로 모든 광고를 동일 기준으로 평가하기 어렵습니다.
- 보상금액은 클릭과 완료에 영향을 줄 수 있으나, 높은 보상이 항상 효율적인 반응으로 이어지는 것은 아닙니다.
- 시간대와 요일에 따라 클릭 수와 완료율이 다르게 나타나므로 운영 스케줄 최적화 여지가 있습니다.
- BERTopic으로 재분류한 행동유형 기준 분석은 기존 ads_type보다 실제 유저 행동 차이를 더 잘 드러낼 수 있습니다.

---

## 3-2. 수익성 분석

수익성은 광고 반응이 실제 플랫폼 마진으로 이어지는지를 보는 축입니다.

### 주요 지표

| 지표 | 설명 |
|---|---|
| `total_valid_margin` | 광고별 유효 총마진 |
| `total_valid_margin_per_click` | 클릭당 마진 |
| `total_valid_margin_per_complete` | 완료당 마진 |
| `negative_margin_pct` | 음수 마진 비율 |

### 주요 분석 항목

- 광고별 총마진 요약
- 광고유형별 완료당 마진 차이
- reward_band별 수익성 차이
- 마진 상위 광고 분석
- 마진이 특정 광고나 매체에 집중되는지 확인
- 음수 마진 및 재무 이상치 점검

### 핵심 해석

- 반응성이 높아도 마진이 낮으면 운영 우선순위가 높다고 보기 어렵습니다.
- 완료율이 높더라도 리워드 비용이 크면 수익성이 낮아질 수 있습니다.
- 수익성은 CVR과 별도로 반드시 독립 평가축으로 포함되어야 합니다.

---

## 3-3. 안정성 분석

안정성은 광고 성과가 일시적인 반응인지, 일정 기간 동안 유지되는지를 확인하는 축입니다.

### 주요 지표

| 지표 | 설명 |
|---|---|
| `observed_n_day` | 광고 관측일차 |
| `active_days` | 실제 클릭이 발생한 활성 일수 |
| `active_day_ratio` | 관측 기간 대비 활성 일수 비율 |
| `delta_stage_cvr` | 초기/후기 CVR 변화 |
| `delta_stage_click` | 초기/후기 클릭 수 변화 |
| `fail_ratio` | 저성과 또는 실패 일수 비율 |

### 주요 분석 항목

- 광고 관측일차별 평균 클릭 수 / 완료 수 / CVR
- active_days 구간별 광고 수 분포
- stage 기반 초기/후기 성과 변화
- 꾸준한 광고와 반짝형 광고 구분
- 성과 변동성과 실패 안정성 분석
- 광고별 지속성 및 관측 신뢰도 판단

### 핵심 해석

- 초반 관측일의 성과가 높게 나타나는 경향이 있었습니다.
- 일부 광고는 초반 반응은 강하지만 이후 성과가 급격히 감소할 수 있습니다.
- 관측일 수가 짧거나 클릭 수가 부족한 광고는 안정성 점수를 강하게 부여하기 어렵습니다.
- 따라서 안정성은 충분한 관측 기간과 표본 수를 전제로 해석해야 합니다.

---

## 3-4. 규모 분석

규모는 광고가 전체 운영 성과에 미치는 영향력을 확인하는 보조 축입니다.

### 주요 분석 항목

- 광고별 클릭 수 상위 10개
- 광고별 완료 수 상위 10개
- 상위 10 / 50 / 100개 광고의 클릭 비중
- 클릭 수 및 완료 수 로렌츠 곡선
- HHI 지수
- 성과 집중도 분석

### 핵심 해석

- 일부 광고가 전체 클릭과 완료의 큰 비중을 차지할 경우, 플랫폼 성과가 특정 광고에 의존할 수 있습니다.
- 규모는 독립적인 우열 판단 기준이라기보다 반응성·수익성·안정성이 실제 운영에 미치는 영향력을 보정하는 축으로 해석했습니다.

---

# 4. 광고 점수화 데이터마트 설계

## 데이터 누수 방지

모델링 데이터마트에서는 최종 성과 기반 지표가 학습 과정에 섞이지 않도록 누수 방지 구조를 설계했습니다.

### 주요 변경사항

기존 방식에서는 전체 데이터 기준으로 percentile rank를 계산하면서 val/test 분포 정보가 train 점수 계산에 섞일 수 있었습니다.  
이를 방지하기 위해 다음 구조로 변경했습니다.

```text
1. 유효 광고 선정
2. raw 변수 가공
3. train / val / test split
4. train 기준 통계량 fit
5. val/test는 train 분포에 끼워 넣어 percentile transform
6. train 기준 threshold로 라벨 생성
```

### GroupedECDFScorer

광고 유형과 보상 구간별 분포 차이를 반영하기 위해 `GroupedECDFScorer`를 도입했습니다.

- train 데이터에서 그룹별 경험적 분포(ECDF)를 fit
- val/test 데이터는 train 분포 위에서 상대 위치만 계산
- 그룹 표본이 부족하면 전역 train 분포로 fallback
- 전역 표본도 부족하면 중립값 0.5 부여

## 품질 점수 산식

광고 품질 점수는 다음 네 가지 성과 축을 train 기준 percentile 점수로 변환한 뒤 가중합으로 계산했습니다.

| 구성 요소 | 가중치 |
|---|---:|
| 마진 점수 (`score_margin`) | 0.35 |
| CVR 점수 (`score_cvr`) | 0.30 |
| 완료 규모 점수 (`score_complete`) | 0.20 |
| CTIT 점수 (`score_ctit`) | 0.15 |

```text
quality_score =
    0.35 * score_margin
  + 0.30 * score_cvr
  + 0.20 * score_complete
  + 0.15 * score_ctit
```

최종 점수는 0~100 범위로 변환했습니다.

---

# 5. Model 1: 광고 고성과 예측 + 점수화 + 등급화

## 모델 정의

광고 등록 시점에 확보 가능한 속성 정보만으로 최종 품질 점수 상위 25%에 해당하는 고성과 광고를 사전 예측하는 이진 분류 모델입니다.

### 적용 대상

- 유효 클릭 수 10건 이상 광고
- 광고 등록 시점에 알 수 있는 속성 정보 중심
- 사후 성과 기반 이상치 피처는 모델 피처로 사용하지 않음

## 피처 설계

### 피처 후보

- 수치형 광고 속성
- One-Hot Encoding된 광고 유형 / 카테고리 / 보상 구간
- TF-IDF 기반 광고명/설명 텍스트 피처

### 최종 피처 선택

최고 단일 성능보다 운영 안정성, 해석성, 단순성을 우선했습니다.

최종 선택:

```text
B_num+ohe
```

- 피처 수: 48개
- 수치형 + OHE 조합
- TF-IDF는 성능이 높았지만 운영 부담과 피처 수 증가를 고려해 제외

## 최종 모델

```text
[full] XGBoost
```

### 최종 성능

| 지표 | 값 |
|---|---:|
| Validation AUC | 0.8094 |
| Test AUC | 0.8347 |
| Test PR-AUC | 0.6221 |
| Train-Test Gap | 0.0777 |
| 목표 AUC 0.75 | 달성 |

### 이상치 제외 민감도 분석

| 지표 | 값 |
|---|---:|
| Clean Test AUC | 0.8279 |
| Clean Test PR-AUC | 0.6132 |

full 대비 성능 하락이 크지 않아, 최종 모델의 성능이 이상치에만 의존한다고 보기 어렵습니다.

## 광고 등급화 결과

예측 확률을 기준으로 광고를 S/A/B/C/D 등급으로 나누었습니다.

| 등급 | 건수 | 비중 | 실제 고성과 비율 |
|---|---:|---:|---:|
| S | 721 | 19.9% | 68.2% |
| A | 724 | 20.0% | 34.7% |
| B | 761 | 21.0% | 14.6% |
| C | 711 | 19.6% | 3.2% |
| D | 711 | 19.6% | 0.7% |

S등급에서 실제 고성과 비율이 가장 높고, D등급으로 갈수록 고성과 비율이 낮아지는 구조가 확인되었습니다.  
따라서 모델 예측 확률을 광고 운영 우선순위화에 사용할 수 있습니다.

## Model 1 최종 하이퍼파라미터

```json
{
  "n_estimators": 643,
  "learning_rate": 0.22329916619526158,
  "max_depth": 8,
  "subsample": 0.9247734590358337,
  "colsample_bytree": 0.6795292369167751,
  "gamma": 1.0524097192151365,
  "reg_alpha": 0.00044108827063748764,
  "reg_lambda": 0.10506932611511502,
  "min_child_weight": 4
}
```

---

# 6. Model 2: 광고 조기부진 예측

## 모델 정의

광고 시작 후 D+3 시점에 확보 가능한 등록 속성과 초기 클릭 실적을 활용하여 최종 품질 점수 하위 25%에 해당하는 부진 광고를 조기에 탐지하는 이진 분류 모델입니다.

### 적용 대상

- `early_click >= 10` 광고만 ML 예측 대상으로 포함
- `early_click < 10` 광고는 관측 부족으로 판단 보류
- 자동 중단 모델이 아니라 운영자 점검 우선순위화 보조 모델

## 누수 피처 제거

초기 실적을 사용하는 모델이기 때문에 누수 가능성을 특히 주의했습니다.

완료수 계열 피처는 최종 부진 라벨과 직접 연결될 수 있으므로 제거했습니다.

제거 대상 예시:

- `early_complete`
- `complete_day1`
- `complete_day2`
- `complete_day3`

## 최종 모델

```text
[clean] RandomForest
```

### 최종 성능

| 지표 | 값 |
|---|---:|
| CV PR-AUC | 0.5028 |
| Validation PR-AUC | 0.6019 |
| Test PR-AUC | 0.5479 |
| CV-Test PR-AUC 차이 | -0.0451 |
| Test ROC-AUC | 0.8269 |
| Train-Test ROC-AUC Gap | 0.0905 |
| Test Precision (threshold=0.5) | 0.4218 |
| Test Recall (threshold=0.5) | 0.7381 |
| Test F1 (threshold=0.5) | 0.5368 |

## 운영 threshold 후보

| 기준 | Threshold | Test Precision | Test Recall | Test F1 |
|---|---:|---:|---:|---:|
| F1 최적 | 0.5885 | 0.4796 | 0.5595 | 0.5165 |
| Recall ≥ 0.70 | 0.4637 | 0.4048 | 0.8095 | 0.5397 |
| Recall ≥ 0.80 | 0.3969 | 0.3717 | 0.8452 | 0.5164 |

## 해석상 주의

Model 2는 PR-AUC 기준 train-val gap이 남아 있어 잔존 과적합 가능성이 있습니다.  
다만 ROC-AUC 기준 train-test gap과 CV-test PR-AUC 차이는 상대적으로 안정적으로 나타났습니다.

따라서 이 모델은 다음 용도로 제한하는 것이 적절합니다.

- 자동 중단 판단 X
- 광고 운영자 점검 우선순위화 O
- 부진 가능성이 높은 광고를 먼저 확인하는 조기 경보 보조 지표 O

## Model 2 최종 하이퍼파라미터

```json
{
  "n_estimators": 490,
  "max_depth": 8,
  "min_samples_leaf": 13,
  "max_features": 0.36340924446823
}
```

---

# 7. 주요 산출물

## 1. 광고 재분류 결과

- `final_media`
- `final_action`
- `media_source`
- `action_source`
- rule evidence
- topic candidate
- low confidence review sample

## 2. EDA 산출물

- 전체 퍼널 요약
- 광고유형별 성과 요약
- 보상구간별 반응성 분석
- 시간대별 클릭 / CVR / CTIT 분석
- 광고별 마진 요약
- 안정성 평가 테이블
- 광고 성과 집중도 분석
- 운영 후보 분류 테이블

## 3. 모델링 산출물

- `ad_mart_v5.parquet`
- `model1_train.parquet`
- `model1_val.parquet`
- `model1_test.parquet`
- `model2_train.parquet`
- `model2_val.parquet`
- `model2_test.parquet`
- `preprocessing_pipeline.joblib`
- `model1_best_params.json`
- `model2_best_params.json`

---

# 8. 프로젝트 구조

```text
reward-ad-performance-scoring/
├── README.md
├── requirements.txt
├── notebooks/
│   ├── 01_01_ad_classification_bertopic.ipynb
│   ├── 01_02_preprocessing.ipynb
│   ├── 02_EDA.ipynb
│   ├── 03_00_ML_EDA.ipynb
│   ├── 03_01_ML_mart.ipynb
│   ├── 03_02_ML1.ipynb
│   └── 03_03_ML2.ipynb
├── params/
│   ├── model1_best_params.json
│   └── model2_best_params.json
├── images/
│   ├── funnel_summary.png
│   ├── response_by_type.png
│   ├── margin_analysis.png
│   ├── stability_curve.png
│   ├── concentration_lorenz.png
│   ├── model1_grade_result.png
│   └── model2_threshold_result.png
└── .gitignore
```

---

# 9. 실행 방법

## 패키지 설치

```bash
pip install -r requirements.txt
```

## 노트북 실행 순서

```text
1. notebooks/01_01_ad_classification_bertopic.ipynb
2. notebooks/01_02_preprocessing.ipynb
3. notebooks/02_EDA.ipynb
4. notebooks/03_00_ML_EDA.ipynb
5. notebooks/03_01_ML_mart.ipynb
6. notebooks/03_02_ML1.ipynb
7. notebooks/03_03_ML2.ipynb
```

---

# 10. requirements 예시

```text
pandas
polars
numpy
matplotlib
seaborn
scikit-learn
xgboost
lightgbm
optuna
shap
joblib
pyarrow
openpyxl
bertopic
sentence-transformers
umap-learn
hdbscan
tqdm
```

---

# 11. GitHub 업로드 시 제외 권장 파일

데이터 파일은 용량이 크고 내부 운영 데이터가 포함될 수 있으므로 업로드하지 않는 것을 권장합니다.

## `.gitignore` 예시

```gitignore
# data
*.csv
*.xlsx
*.xls
*.parquet
*.pkl
*.joblib

# model artifacts
models/
outputs/
reports/

# notebook checkpoints
.ipynb_checkpoints/

# OS files
.DS_Store
._*
Thumbs.db

# environment
.env
venv/
.venv/
__pycache__/
```

---

# 12. 프로젝트 결과 요약

## 핵심 결과

- 리워드 광고 성과를 클릭 수나 CVR 단일 지표가 아닌 반응성, 수익성, 안정성, 규모의 다축 구조로 재정의했습니다.
- 전체 퍼널 기준 클릭 수 3,181,777건, 완료 수 1,383,784건, CVR 43.49%를 확인했습니다.
- 기존 광고 유형이 매체 기준과 행동 기준이 혼재되어 있음을 확인하고, BERTopic + Rule-based Cascade 방식으로 `final_media`, `final_action`을 재분류했습니다.
- 광고 품질 점수 산식에 마진, CVR, 완료 규모, CTIT를 반영했습니다.
- Model 1은 고성과 광고 예측에서 Test AUC 0.8347을 기록했고, S등급 광고의 실제 고성과 비율은 68.2%로 나타났습니다.
- Model 2는 D+3 조기부진 예측에서 Test ROC-AUC 0.8269, Test Recall 0.7381을 기록했으며, 운영자 점검 우선순위화 보조 모델로 활용 가능성을 확인했습니다.

## 운영 활용 방안

| 활용 영역 | 설명 |
|---|---|
| 광고 사전 심사 | 등록 시점 속성 기반으로 고성과 가능성 예측 |
| 광고 등급화 | S/A/B/C/D 등급으로 운영 우선순위 분류 |
| 조기 경보 | D+3 기준 부진 가능 광고 탐지 |
| 운영 점검 | 부진 가능성이 높은 광고를 운영자가 우선 확인 |
| 광고 유형 관리 | 기존 ads_type 대신 final_media/final_action 기반으로 성과 비교 |
| 리워드 정책 | 보상구간별 반응성·수익성 차이를 기반으로 보상 전략 조정 |

---

# 13. 한계 및 개선 방향

## 한계

- 분석 기간이 한 달치 데이터에 한정되어 장기 계절성까지 일반화하기 어렵습니다.
- Model 2는 PR-AUC 기준 잔존 과적합 가능성이 있어 자동 의사결정 모델로 사용하기에는 제한이 있습니다.
- BERTopic 기반 재분류는 최종 자동 라벨러가 아니라 검수 보조 도구로 보는 것이 안전합니다.
- 데이터 및 결과물 일부는 내부 운영 데이터 성격이 있어 GitHub에 직접 공개하기 어렵습니다.

## 개선 방향

- 여러 달 데이터로 모델 안정성 재검증
- 광고 유형별 별도 threshold 적용
- 조기부진 모델의 표본 수 확대
- 광고 라이프사이클 기반 생존분석 도입
- 리워드 금액 최적화 모델 추가
- Streamlit 대시보드 구축
- 운영자 피드백을 반영한 라벨 재학습 구조 설계

---

# 14. 프로젝트에서 강조할 수 있는 역량

- 복잡한 로그 데이터를 광고 단위 분석 테이블로 재구성
- 클릭, 완료, 수익, 안정성, 규모를 통합한 성과 평가 체계 설계
- 데이터 누수 가능성을 인식하고 train 기준 ECDF 변환 방식으로 보정
- 단순 성능보다 운영 가능성, 해석성, 안정성을 고려한 모델 선택
- BERTopic을 맹목적 자동 분류기가 아닌 검수 보조 도구로 활용
- 모델 결과를 실제 운영 의사결정 구조로 연결
