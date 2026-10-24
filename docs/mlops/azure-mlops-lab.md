---
title: "MLOps 실습: 학습부터 추론까지 연결하기"
date: 2026-10-25
tags: [MLOps, Azure, MLflow, Feast, Kubeflow]
---

## 한눈에 보기 {#reading-map}

**목표: 학습한 모델과 입력의 출처를 남기고, 같은 절차를 다시 실행하기.** 아래 순서로 읽으면 된다.

```mermaid
flowchart LR
    A["① 탐색·학습·평가<br/>JupyterLab"] --> B["② 작업 연결·재실행<br/>Kubeflow"]
    B --> C["③ 피처·모델 준비<br/>Feast · SeaweedFS"]
    C --> D["④ 예측·검증<br/>작업 Pod"]
    D -.->|"개선"| A
```

| 실행 | 기록·비교 | 파일 보관 |
|---|---|---|
| 노트북 커널 / KFP 작업 Pod | MLflow | SeaweedFS |

**MLOps = 데이터 → 학습 → 평가 → 추론 → 개선을 반복·추적하는 체계.** 코드뿐 아니라 데이터·모델의 변화와 품질도 관리한다.

??? note "책 1~5장과 이번 실습"

    | 장 | 핵심 |
    |---|---|
    | 1 | ML 개발·운영 생애주기와 역할 |
    | 2 | DevOps와의 차이, 반복과 자동화 수준 |
    | 3 | Docker·Kubernetes, CI/CD·모니터링 |
    | 4 | MLflow 실험 추적·Feast 피처 관리 |
    | 5 | Kubeflow 파이프라인 |

    『Machine Learning Platform Engineering』의 개념을 소득 분류 예제로 연결했다.

## 실행 환경과 역할 {#_2}

| 환경 | 구성 |
|---|---|
| Azure VM | Ubuntu 22.04 · 8 vCPU / 32 GiB · CPU 학습 |
| VM의 Docker | JupyterLab · Redis |
| kind Kubernetes | KFP standalone · MLflow · SeaweedFS · 작업 Pod |

```mermaid
flowchart LR
    J["Jupyter<br/>Python 커널"] -->|"SDK"| M
    subgraph K8s["Kubernetes"]
        K["KFP API·Controller"] --> P["작업 Pod"]
        P -->|"SDK"| M["MLflow 서버"]
        M --> DB[("기록 DB · PVC")]
        P <-->|"직접 전송"| S[("SeaweedFS")]
    end
    J <--> S
```

| 도구 | 역할 |
|---|---|
| JupyterLab | 코드를 편집하고 셀 실행. 실제 연산은 커널 |
| MLflow SDK → 서버 | Run·설정·점수·모델 정보 기록 → UI에서 비교 |
| KFP SDK → API·Controller → Pod | DAG 정의 → 실행 제어 → Python 작업 실행 |
| SeaweedFS | 모델·피처·산출물 파일 저장 |

Redis는 노트북의 Feast 온라인 조회에 사용했다. 접속은 SSH 터널, 종료는 VM 할당 해제이며 디스크 비용은 남는다.

## ① 노트북에서 실험하기 {#1-mlflow}

```text
CSV → EDA → 분할 → 전처리 → 학습 → 평가 → 설정 변경
```

| 개발자의 작업 | 이번 실습 |
|---|---|
| 문제 정의 | 직업·교육 등 8개 피처로 소득 >50K 여부 예측 |
| 데이터 탐색 | 30,162행의 표·그래프 확인 |
| 입력 준비 | 학습 80% / 테스트 20%, 원핫 인코딩 |
| 모델 비교 | Decision Tree · Random Forest · XGBoost |
| 기록 | 파라미터·점수·분석 산출물·모델 |

핵심 코드만 발췌했다. `fit()`은 학습, `log_*()`는 기록이다. 셀 출력 전체가 자동 저장되지는 않는다.

```python
with mlflow.start_run(run_name=model_name):
    mlflow.log_param('seed', 42)
    model.fit(X_train, y_train)
    logged = mlflow.sklearn.log_model(model, name='model')
```

| 모델 | Accuracy | ROC-AUC | F1 |
|---|---:|---:|---:|
| Decision Tree | 0.8228 | 0.8298 | 0.6103 |
| Random Forest | 0.8276 | 0.8631 | 0.6179 |
| XGBoost | 0.8314 | 0.8818 | 0.6198 |

<figure class="lab-shot" markdown="span">

[![MLflow · 세 모델 비교. 이미지는 클릭하면 원본으로 열린다.](assets/training-comparison.png)](assets/training-comparison.png)

<figcaption>MLflow · 세 모델 비교. 이미지는 클릭하면 원본으로 열린다.</figcaption>
</figure>

`Experiment`는 실험 묶음, `Run`은 개별 실행이다. 모델 선택은 개발자가 한다.
이번에는 테스트 점수로 모델까지 비교했지만, 다음에는 **검증셋으로 선택 → 테스트셋으로 최종 평가**를 분리한다.

## ② 반복 실행으로 옮기기 {#2-kubeflow}

```mermaid
flowchart LR
    A["노트북 셀"] --> B["Python 함수"] --> C["KFP 컴포넌트"] --> D["DAG·YAML"] --> E["작업 Pod"]
```

학습은 **준비 → 학습·모델 저장 → 평가 → 비교**로 분리했다. 자원 부담을 줄여 세 모델을 순차 실행했다.
아래는 추론 DAG의 실제 연결 코드다. 환경·자원 설정은 생략했다.

```python
@dsl.pipeline(name='chapter5-income-batch-inference')
def pipeline(requests_bucket: str, requests_key: str,
             model_uri: str, output_prefix: str = 'predictions'):
    read = read_requests(bucket=requests_bucket, key=requests_key)
    features = retrieve_features(requests=read.outputs['requests'])
    predictions = run_inference(
        features=features.outputs['features'],
        lineage=features.outputs['lineage'], model_uri=model_uri,
        kfp_run_id=dsl.PIPELINE_JOB_ID_PLACEHOLDER,
    )
    write_predictions(
        predictions=predictions.outputs['predictions'],
        bucket=requests_bucket, prefix=output_prefix,
        kfp_run_id=dsl.PIPELINE_JOB_ID_PLACEHOLDER,
    )
```

| 코드 | DAG |
|---|---|
| 컴포넌트 호출 | 파란 작업 노드 |
| `Output` | 노란 산출물 노드 |
| `outputs[...]`를 다음 입력으로 전달 | 의존성 화살표 |

파이프라인 함수는 컴파일 시 그래프를 만든다. 실행 시 핵심 코드는 각 작업 Pod에서 돈다.
재실행은 **Clone run → 파라미터 확인 → Start**. Pod의 기록 코드는 계속 MLflow를 사용한다.

??? example "학습 파이프라인의 비교 리포트"

    <figure class="lab-shot" markdown="span">

    [![KFP 산출물 · 세 모델의 평가 결과와 MLflow Run 링크](assets/kubeflow-training-report.png)](assets/kubeflow-training-report.png)

    <figcaption>KFP 산출물 · 세 모델의 평가 결과와 MLflow Run 링크</figcaption>
    </figure>

## ③ 피처와 모델 준비하기 {#3-feast}

| 대상 | 요청 | 반환 |
|---|---|---|
| Feast 과거 조회 | 사용자 ID + 시점 + 피처 목록 | 해당 시점까지 가장 최근의 유효한 값 |
| Feast 온라인 조회 | 사용자 ID + 피처 목록 | Redis에 적재한 최신 값 |
| MLflow 모델 조회 | Logged Model ID | 모델 정보·저장 위치 |
| SeaweedFS 다운로드 | S3 모델 경로 | 모델 파일 |

`Entity`는 조회 키, `FeatureView`는 소스·피처·시점 정의다. 학습은 CSV를 직접 읽었고, 배치 추론에서 Feast를 사용했다.

```python
result = store.get_historical_features(
    entity_df=request_df,  # 사용자 ID·시점 목록
    features=FEATURES,     # 이번에는 8개 피처
).to_df()
```

```python
metadata = MlflowClient().get_logged_model(model_uri[len('models:/'):])
with tempfile.TemporaryDirectory() as temporary:
    directory = mlflow.artifacts.download_artifacts(
        artifact_uri=metadata.artifact_location,  # s3://...
        dst_path=temporary,
    )
    model = mlflow.sklearn.load_model(directory)
```

**모델 파일은 작업 Pod가 SeaweedFS에서 직접 내려받는다.**

<figure class="lab-shot" markdown="span">

[![SeaweedFS · model.pkl, MLmodel, 실행 환경 파일](assets/seaweedfs-model-files.png)](assets/seaweedfs-model-files.png)

<figcaption>SeaweedFS · model.pkl, MLmodel, 실행 환경 파일</figcaption>
</figure>

??? example "피처 파일과 모델의 학습 출처"

    <figure class="lab-shot" markdown="span">

    [![원본 Parquet 파일 3개](assets/seaweedfs-feature-files.png)](assets/seaweedfs-feature-files.png)

    <figcaption>원본 Parquet 파일 3개</figcaption>
    </figure>

    <figure class="lab-shot" markdown="span">

    [![MLflow Source run · 이 모델을 만든 학습 실행](assets/mlflow-trained-model.png)](assets/mlflow-trained-model.png)

    <figcaption>MLflow Source run · 이 모델을 만든 학습 실행</figcaption>
    </figure>

## ④ 예측하고 검증하기 {#4}

<figure class="lab-shot" markdown="span">

[![KFP · 요청 읽기 → Feast 조회 → 예측 → 결과 저장](assets/kubeflow-inference.png)](assets/kubeflow-inference.png)

<figcaption>KFP · 요청 읽기 → Feast 조회 → 예측 → 결과 저장</figcaption>
</figure>

| 작업 | 산출물 |
|---|---|
| 요청 읽기 | 사용자 ID·시점 5건: `requests` |
| Feast 조회 | 8개 피처와 재현 정보: `features`, `lineage` |
| 모델 예측 | 예측값: `predictions` |
| 결과 저장 | `predictions.parquet`, 저장 확인 정보, HTML 리포트 |

추론 작업의 상한은 CPU 1개·메모리 2 GiB. 모델 ID·피처 목록·Feast 정의·데이터 해시를 MLflow에 기록했다.
해시는 읽은 파일에서 자동 계산한다. 파일 변경을 구분하기 위한 실습 설계다.

??? example "MLflow 추론 기록과 S3 결과 파일"

    <figure class="lab-shot" markdown="span">

    [![사용한 모델 ID·S3 경로를 추론 Run에 기록](assets/inference-model-reference.png)](assets/inference-model-reference.png)

    <figcaption>사용한 모델 ID·S3 경로를 추론 Run에 기록</figcaption>
    </figure>

    <figure class="lab-shot" markdown="span">

    [![Run별 경로의 predictions.parquet](assets/seaweedfs-prediction-file.png)](assets/seaweedfs-prediction-file.png)

    <figcaption>Run별 경로의 predictions.parquet</figcaption>
    </figure>

### 공백 오류 해결 {#_3}

```mermaid
flowchart LR
    A["Private<br/>학습 입력"] --> C["범주 불일치"]
    B["앞 공백 + Private<br/>추론 입력"] --> C
    C --> D["미등록 범주 무시<br/>입력 정보 손실"] --> E["서로 다른 요청<br/>같은 예측 확률"]
```

원본·학습 코드를 대조해 공백 차이를 찾았다. `handle_unknown='ignore'` 때문에 오류 없이 진행됐다.

```python
X = df[COLUMNS].apply(lambda column: column.str.strip())
labels = model.predict(X)
probabilities = model.predict_proba(X)[:, list(model.classes_).index(1)]
```

| 수정 | 검증 |
|---|---|
| 모델 유지, 추론에 공백 제거 적용 | 5건 모두 원핫 범주 8개 활성화 |
| 정규화한 입력도 MLflow에 저장 | 미등록 범주 수 0 |
| 같은 모델로 별도 계산 | 파이프라인 예측 클래스·확률과 일치 |

<figure class="lab-shot" markdown="span">

[![수정 후 예측 결과 · 책의 공개 예제 데이터](assets/prediction-result.png)](assets/prediction-result.png)

<figcaption>수정 후 예측 결과 · 책의 공개 예제 데이터</figcaption>
</figure>

별도 정답 대조는 **3/5 일치**. 학습에 포함된 소수 샘플이며, 일반화 성능이나 MLflow에 자동 기록된 정확도가 아니다.

## 결론과 다음 실험 {#_4}

**실행 성공 ≠ 올바른 입력 ≠ 좋은 예측.** 이번에는 출처를 추적하고 같은 절차를 다시 실행할 기반을 만들었다.

| 다음 작업 | 목적 |
|---|---|
| 학습·검증·테스트 분리 | 모델 선택과 최종 평가 구분 |
| 전처리 통합 | 학습·추론 입력 일치 |
| 데이터·피처·파라미터 변경 | MLflow에서 개선 효과 비교 |
| CI/CD·운영 모니터링 | 코드 검사·이미지 빌드·배포, 오류·지연·품질 관측 |

```mermaid
flowchart LR
    A["오분류 분석"] --> B["데이터·설정 변경"] --> C["KFP 재실행"] --> D["MLflow 비교"]
    D --> A
```

운영 API·모니터링·자동 재학습은 다음 범위다. 책의 GitLab CI·Argo CD·Prometheus·Grafana는 이 확장에 해당한다.
블로그 자동 배포와 ML 파이프라인의 CI/CD는 별개다.

??? note "참고"

    - 책 예제: [4장](https://github.com/practical-mlops/chapter-4), [5장](https://github.com/practical-mlops/chapter-5)
    - [JupyterLab](https://jupyterlab.readthedocs.io/en/stable/user/notebook.html) · [MLflow](https://mlflow.org/docs/latest/ml/tracking/) · [KFP](https://www.kubeflow.org/docs/components/pipelines/overview/)
