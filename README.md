# COVID-19 사망 위험 예측

멕시코 보건부의 COVID-19 환자 데이터를 바탕으로 사망 위험을 예측하고, 불균형 데이터에서 모델·리샘플링·분류 임계값이 성능에 미치는 영향을 비교한 개인 통계데이터사이언스 프로젝트입니다. 데이터 전처리, 코호트 구성, 모델 비교, 리샘플링과 임계값 분석을 전 과정 수행했습니다.

## Highlights

- 전체 환자 1,048,575명과 COVID-19 확진자 391,979명을 별도 코호트로 분석했습니다.
- Logistic Regression과 LightGBM을 비교하고 언더샘플링·오버샘플링 효과를 함께 검토했습니다.
- 검증 데이터에서 F1 기준 임계값을 선택했으며, 테스트 데이터는 임계값 선택에 사용하지 않았습니다.
- 전체 환자 코호트의 최종 LightGBM은 테스트 ROC-AUC `0.9332`, PR-AUC `0.5610`을 기록했습니다.

## 프로젝트 개요

의료 데이터처럼 사망 사례가 적은 불균형 데이터에서는 정확도만으로 모델을 평가하기 어렵습니다. 이 프로젝트는 다음 질문에 답하는 것을 목표로 합니다.

- 선형 모델과 비선형 모델의 판별 성능은 어떻게 다른가?
- 전체 환자와 확진자 코호트에서 결과가 어떻게 달라지는가?
- 리샘플링과 분류 임계값 조정 중 어떤 방법이 실제 예측 지표에 더 큰 영향을 주는가?
- Recall과 Precision 사이의 선택이 의료 의사결정 관점에서 어떤 의미를 갖는가?

## 데이터

- 출처: [COVID-19 Dataset on Kaggle](https://www.kaggle.com/datasets/meirnizri/covid19-dataset) — 멕시코 보건부 공개 데이터
- 전체 환자 코호트: 1,048,575명
- 확진자 코호트: 391,979명
- 예측 대상: `DATE_DIED`를 바탕으로 생성한 사망 여부 `DEATH`
- 주요 변수: 나이, 성별, 폐렴, 당뇨, 고혈압, 비만, 흡연 여부와 기타 기저질환

원본 데이터의 `CLASIFFICATION_FINAL`은 오타가 아니라 실제 컬럼명입니다. 전체 환자 분석에서는 이 변수를 포함하고, 확진자 코호트에서는 입력 변수에서 제외했습니다.

주요 전처리는 다음과 같습니다.

- 질환 변수에서는 특수 코드 `98`을 `Unknown`으로 처리했습니다.
- 임신 여부에서는 `97`, `98`, `99`를 `Unknown`으로 묶고, 전체 값을 `NotApplicable`, `No`, `Yes`, `Unknown`으로 정리했습니다.
- `INTUBED`, `ICU`, `PATIENT_TYPE`은 내생 변수로 보고 입력에서 제외했습니다.
- 범주형 변수는 원-핫 인코딩하고, 두 모델 모두 `AGE`를 표준화했습니다.

## 실험 설계

| 항목 | 설정 |
| --- | --- |
| 코호트 | 전체 환자와 COVID-19 확진자를 각각 분석했습니다. |
| 데이터 분할 | 계층화 분할로 Train 60%, Validation 20%, Test 20%를 구성했습니다. |
| 모델 | Logistic Regression과 LightGBM을 비교했습니다. |
| 불균형 대응 | 원본 분포와 Random Under-Sampling, Random Over-Sampling을 비교했습니다. |
| 모델 선택 | 반복 실행 중 Validation ROC-AUC 기준 |
| 임계값 선택 | Validation F1 기준 |
| 평가 지표 | ROC-AUC, PR-AUC, Accuracy, Precision, Recall, F1, Confusion Matrix |

선택된 실행의 Validation ROC-AUC는 다음과 같습니다. 이 값은 최종 테스트 성능과 구분됩니다.

| 코호트 | Logistic Regression | LightGBM |
| --- | ---: | ---: |
| 전체 환자 | 0.9275 | **0.9337** |
| 확진자 | 0.9073 | **0.9089** |

## 주요 결과

최종 LightGBM의 테스트 결과입니다. 최적 임계값은 Validation 데이터에서 정한 뒤 Test 데이터에 그대로 적용했습니다. ROC-AUC와 PR-AUC는 예측 순위의 품질을, Recall은 사망자 누락을 줄이는 정도를, Precision은 불필요한 경고를 줄이는 정도를 나타냅니다. F1은 Validation 데이터에서 임계값을 선택하는 기준으로 사용했습니다.

| 코호트 | 임계값 | ROC-AUC | PR-AUC | Accuracy | Precision | Recall | F1 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 전체 환자 | 0.500 | 0.9332 | 0.5610 | 0.8469 | 0.3095 | **0.8824** | 0.4583 |
| 전체 환자 | 0.825 | 0.9332 | 0.5610 | **0.9243** | **0.4882** | 0.6560 | **0.5598** |
| 확진자 | 0.500 | 0.9120 | 0.5944 | 0.8214 | 0.4287 | **0.8748** | 0.5754 |
| 확진자 | 0.700 | 0.9120 | 0.5944 | **0.8691** | **0.5185** | 0.7529 | **0.6141** |

- 임계값을 높이면 거짓 양성(False Positive)이 줄어 Precision과 F1이 개선되는 대신 Recall이 낮아집니다.
- ROC-AUC와 PR-AUC는 예측 순위에 기반한 지표이므로 임계값 변경 전후 값이 같습니다.
- 언더샘플링과 오버샘플링은 이 실험에서 성능 차이가 작았고, 최종 분류 결과에는 임계값 조정이 더 큰 영향을 주었습니다.

따라서 의료 상황에서는 하나의 임계값을 정답으로 제시하기보다, 사망자 누락을 줄이는 Recall 중심 정책과 불필요한 경고를 줄이는 Precision 중심 정책 사이에서 목적에 맞게 선택해야 합니다.

## 저장소 구성

```text
data/
├── raw/                         # Kaggle 원본 데이터
└── processed/                   # 전체 환자·확진자 전처리 데이터
notebooks/
├── eda.ipynb                    # 탐색적 데이터 분석
├── preprocessing.ipynb          # 전처리 및 코호트 생성
├── modeling_logistic.ipynb      # Logistic Regression 실험
├── modeling_lgbm.ipynb          # LightGBM 실험과 최종 테스트 결과
└── modeling_artifacts/          # 선택 실행의 모델·메타데이터
doc/
└── covid19-analysis-report.pdf  # 프로젝트 보고서
```

## 실행 및 결과 확인

필요한 패키지는 다음과 같이 설치할 수 있습니다.

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install jupyter pandas numpy matplotlib seaborn scikit-learn lightgbm imbalanced-learn joblib
jupyter lab
```

환경을 설치한 뒤 다음 순서로 실행합니다. 탐색적 데이터 분석은 전처리 결과 생성에 필요하지 않으므로 결과만 확인할 때는 건너뛸 수 있습니다.

- [탐색적 데이터 분석](notebooks/eda.ipynb)을 확인합니다.
- [데이터 전처리](notebooks/preprocessing.ipynb)로 두 코호트의 CSV를 생성합니다.
- [Logistic Regression 모델링](notebooks/modeling_logistic.ipynb)을 실행합니다.
- [LightGBM 모델링 및 최종 평가](notebooks/modeling_lgbm.ipynb)를 실행합니다.

노트북에는 실험 당시의 로컬 절대 경로가 남아 있어 재실행 전에 다음 경로를 현재 저장소 기준으로 바꿔야 합니다.

- EDA와 전처리 노트북의 원본 경로를 `data/raw/Covid Data.csv`로 바꿉니다.
- 전처리 노트북의 두 CSV 저장 경로를 `data/processed/` 아래로 바꿉니다.
- 모델링 노트북의 두 데이터 로딩 경로를 `data/processed/` 아래의 해당 CSV로 바꿉니다.

전처리된 CSV가 저장소에 포함되어 있으므로 모델링 결과만 확인할 때는 전처리 단계를 건너뛸 수 있습니다. 저장된 실행 결과와 선택 기준은 `notebooks/modeling_artifacts/`의 JSON 메타데이터에서도 확인할 수 있습니다.

## 한계와 주의사항

- 단일 공개 데이터셋을 사용했으며, 다른 기관이나 시점의 데이터에 대한 외부 검증은 수행하지 않았습니다.
- F1 기준 임계값은 이 데이터 분할에서 선택된 값이며 의료 현장의 최적 임계값을 의미하지 않습니다.
- 데이터 전처리와 모델 평가를 재현하려면 실행 환경과 데이터 경로를 맞춰야 합니다.
- 이 결과물은 교육·연구 목적이며 실제 진단이나 의료 의사결정에 사용할 수 없습니다.

## 관련 자료

- [프로젝트 보고서](doc/covid19-analysis-report.pdf)
- [LightGBM 선택 실행 메타데이터](notebooks/modeling_artifacts/metadata_lgbm.json)
- [확진자 LightGBM 선택 실행 메타데이터](notebooks/modeling_artifacts/metadata_lgbm_positive.json)

## 만든 사람

Jeonghyun Hwang (황정현) — 통계데이터사이언스 과목 프로젝트
