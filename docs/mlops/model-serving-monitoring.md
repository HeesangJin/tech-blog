---
title: "MLOps 실습 2: 모델을 API로 배포하고 관측하기"
date: 2026-11-01
tags: [MLOps, Airflow, Feast, Kubeflow, MLflow, BentoML, Evidently, Gitea]
---

지난 [첫 번째 실습](azure-mlops-lab.md)에서는 소득 분류 모델을 학습하고, 저장한 모델로 배치 추론을 실행했다.
이번에는 **데이터 준비와 학습을 분리하고, 선택한 모델을 API로 배포한 뒤 입력 데이터의 변화까지 확인**했다.
화면 이미지는 누르면 원본 크기로 볼 수 있다.

## 코드 작성부터 운영까지, 진행 순서

실습에서는 노트북과 파이프라인을 먼저 만들고, 나중에 코드를 Gitea로 옮겨 Actions를 붙였다.
아래는 **완성한 구성으로 작업할 때의 순서**다. 실제로 수동 처리한 부분도 함께 적었다.

1. **코드를 개발한다.** JupyterLab에서 데이터를 살펴보고 학습을 시험한다. 반복할 데이터 준비·학습·평가 과정을 Python 파일로 정리하고, API와 입력 분석 코드를 작성한다.

2. **코드를 Gitea에 저장한다.** 학습·평가 코드를 포함해 DAG, API, Dockerfile, 테스트, Actions YAML을 commit·push한다. 데이터와 모델 파일은 SeaweedFS에 둔다.

3. **코드와 라이브러리를 이미지로 묶는다.** 데이터 준비·학습 이미지는 VM에서 수동 빌드해 kind 노드에 넣었다. BentoML·Evidently 이미지는 수동 Actions로 빌드해 Gitea 레지스트리에 올렸다.

4. **데이터 준비·학습 작업을 등록한다.** Airflow에는 DAG 파일을 반영한다. Kubeflow는 SDK 코드를 YAML로 컴파일해 등록한다. 각 작업이 사용할 이미지도 지정한다. 이 등록은 수동으로 했다.

5. **학습할 데이터를 준비한다.** Airflow를 수동 실행해 원본 검사, SeaweedFS 버전별 저장, Feast 정의 등록을 수행한다. 결과로 받은 `manifest_uri`가 학습에 넘길 데이터 버전의 주소다.

6. **학습·평가를 실행한다.** Kubeflow에 `manifest_uri`를 넣고 실행한다. Feast로 피처를 조회해 세 모델을 학습하고, 검증셋으로 고른 모델을 테스트한다. 실행 기록은 MLflow에, 모델 파일은 SeaweedFS에 남긴다.

7. **선택한 모델을 배포한다.** Actions 배포 작업에 `image_tag`와 MLflow의 `comparison_run_id`를 넣고 수동 실행한다. BentoML Pod가 SeaweedFS에서 해당 모델을 받아 로딩하고, 시험 예측을 통과하는지 확인한다.

8. **요청을 받고 입력 변화를 확인한다.** BentoML의 요청·예측 기록을 SeaweedFS에 쌓는다. 기록이 모이면 Airflow에서 Evidently를 수동 실행해 분포·결측값을 비교하고 MLflow에 결과를 남긴다.

**Git에 코드를 올리는 것과 그 코드가 실행 환경에 반영되는 것은 별개다.** 코드를 바꾸면 해당 이미지를 다시 만들고, 작업이나 배포가 새 이미지를 사용하도록 갱신해야 한다.

## 전체 흐름

```mermaid
flowchart LR
    G["코드 개발·저장<br/>Jupyter · Gitea"]
    I["이미지 준비<br/>VM 수동 · Actions"]
    R["작업 등록·실행<br/>Airflow · Kubeflow"]
    S["선택 모델 배포<br/>Actions · BentoML"]
    M["입력 분석<br/>Evidently"]
    G --> I --> R --> S --> M
```

**현재는 단계마다 사람이 결과를 확인하고 다음 작업을 수동 실행한다.**
데이터 버전의 주소는 학습 입력으로, 모델 비교 Run ID는 배포 입력으로 넘긴다.

| 흐름을 받치는 도구 | 역할 |
|---|---|
| JupyterLab | 데이터를 살펴보고 코드를 개발하는 작업 공간 |
| SeaweedFS | 데이터·모델·산출물의 실제 파일을 보관하는 창고 |
| MLflow | 설정·점수·데이터 버전·모델의 연결을 남기는 실험 일지 |
| Gitea + Actions | 코드 보관, 컨테이너 이미지 빌드와 배포 |

실행 환경은 기존 Azure VM과 kind Kubernetes를 계속 사용했다.

## Git에는 무엇을 저장했나

Gitea는 이번 실습의 Git 서버다. 데이터 준비·학습·평가·서빙·분석 코드가 같은 저장소에 있다.
현재 Actions에 연결한 이미지 빌드 범위는 서빙·분석 코드다.

<figure class="lab-shot" markdown="span">

[![Gitea 저장소의 실제 폴더 목록. src, infra, ops, tests와 .gitea/workflows가 보인다](assets/model-serving-monitoring/gitea-source-files.jpg)](assets/model-serving-monitoring/gitea-source-files.jpg)

<figcaption>Gitea · 코드를 저장한 곳. 모델 파일과 학습 데이터는 이 폴더 목록에 없다.</figcaption>
</figure>

| 폴더 | 저장한 것 |
|---|---|
| `src/` | 데이터 준비·학습·평가·분포 분석 Python 코드 |
| `infra/` | Airflow DAG, Kubeflow 파이프라인, BentoML API, Dockerfile |
| `ops/` | 이미지 빌드, Pod 교체, 배포 후 검사 코드 |
| `tests/` | 데이터·API·배포 코드의 테스트 |
| `.gitea/workflows/` | Actions에서 어떤 순서로 빌드하고 배포할지 적은 YAML |

**코드는 Git 저장소, 컨테이너 이미지는 Gitea 레지스트리, 데이터와 모델 파일은 SeaweedFS에 저장했다.**

## ① 데이터 준비: Airflow와 Feast

처음에는 학습 코드에 CSV 읽기와 Feast 등록·조회가 섞여 있었다.
데이터가 바뀔 때마다 다시 준비할 수 있도록 **데이터 준비 작업을 별도 Airflow DAG로 분리**했다. DAG는 작업과 실행 순서를 그린 것이다.

```text
원본 검증 → 버전별 피처 파일·Feast 정의 게시 → 조회 검증·버전 확정
```

<figure class="lab-shot" markdown="span">

[![Airflow에서 데이터 준비 작업 세 개가 모두 성공한 화면](assets/model-serving-monitoring/airflow-data-preparation.jpg)](assets/model-serving-monitoring/airflow-data-preparation.jpg)

<figcaption>Airflow · 왼쪽부터 순서대로 실행했고, 상자 세 개가 모두 초록색 success로 끝났다.</figcaption>
</figure>

오른쪽 위 **트리거**로 수동 실행했다. 화면의 상자는 다음 일을 한다.

| 화면의 상자 | 하는 일 |
|---|---|
| `ingest_inputs` | 원본을 읽고, 파일이 바뀌었는지와 필요한 열이 있는지 확인 |
| `publish_definitions` | 데이터를 버전별로 저장하고 Feast에 파일 위치와 피처 정의 등록 |
| `validate_promote` | Feast로 실제 읽어 본 뒤 사용할 버전을 확정하고 MLflow에 기록 |

**다음 학습으로 넘길 값은 마지막 작업이 돌려준 `manifest_uri`다.**
`validate_promote` 상자를 눌러 로그의 `manifest_uri`를 복사하고, Kubeflow의 같은 이름 입력칸에 붙여넣는다.
사용할 데이터 파일과 Feast 정의가 적힌 JSON 파일의 주소다. 이 파일 형식은 실습에서 직접 정했다.

Airflow와 Kubeflow는 모두 작업을 연결하고 실행한다. 이번에는 두 도구를 익히면서,
데이터 게시와 모델 학습을 독립적으로 실행하기 위해 역할을 나눴다.

## ② 학습·평가: Kubeflow와 MLflow

학습 파이프라인은 세 Task로 정리했다. **Feast 등록은 앞 단계에서 끝나고, 여기서는 선택한 버전을 조회한다.**

| Task | 입력 → 결과 |
|---|---|
| 피처 준비 | 버전 manifest → Feast 조회·정답 결합·데이터 분할 |
| 세 모델 학습 | 학습 데이터 → Decision Tree·Random Forest·XGBoost |
| 평가·비교 | 검증 F1 비교 → 모델 선택 → 선택 모델만 테스트 평가 |

<figure class="lab-shot" markdown="span">

[![Kubeflow에서 피처 준비, 세 모델 학습, 검증 비교와 테스트 평가 작업이 모두 성공한 화면](assets/model-serving-monitoring/kubeflow-training-completed.jpg)](assets/model-serving-monitoring/kubeflow-training-completed.jpg)

<figcaption>Kubeflow · 작업 세 개가 모두 끝났다. dataset은 준비한 데이터, models는 학습한 세 모델이다.</figcaption>
</figure>

첫 글에서는 학습·테스트를 80:20으로 나누고 테스트 점수로 모델까지 비교했다.
이번에는 기존 테스트 행을 유지하고 학습 쪽에서 검증 데이터를 분리했다. 비율은 약 **학습 64% / 검증 16% / 테스트 20%**다.

MLflow에는 각 모델의 설정·점수, 비교 결과, 선택한 모델 ID와 데이터 버전을 연결했다.
모델 파일은 SeaweedFS에 저장한다. 이번에 개선한 것은 **선택·평가 절차**이며, 이 변경만으로 예측 성능이 좋아졌다고 결론 내리지는 않았다.

학습이 끝나면 MLflow에서 **`compare-and-select` 실행을 열고 Overview의 Run ID**를 복사한다. 다음 배포에서 이 값을 사용한다.

<figure class="lab-shot" markdown="span">

[![MLflow compare-and-select 실행의 Run ID와 선택된 학습 Run 연결이 표시된 화면](assets/model-serving-monitoring/mlflow-model-selection.jpg)](assets/model-serving-monitoring/mlflow-model-selection.jpg)

<figcaption>MLflow · 가운데 Run ID를 복사한다. Gitea의 comparison_run_id 칸에 넣을 값이다.</figcaption>
</figure>

## ③ 배포: Gitea Actions와 BentoML

배치 작업이 끝날 때마다 Pod가 종료되는 방식에서, **모델을 메모리에 올려 두고 요청에 응답하는 API**로 확장했다.
BentoML은 모델 로딩과 예측 코드를 API 서비스로 묶는 데 사용했다.

먼저 Gitea의 **Build images (manual)**을 실행해 이미지를 만든다.

<figure class="lab-shot" markdown="span">

[![Gitea Actions 이미지 빌드 11번의 코드 검사, 이미지 빌드, 테스트, 레지스트리 저장이 성공한 화면](assets/model-serving-monitoring/gitea-image-build.jpg)](assets/model-serving-monitoring/gitea-image-build.jpg)

<figcaption>Actions · 빌드 #11. 코드를 검사하고 이미지를 만든 뒤 테스트를 거쳐 레지스트리에 올렸다.</figcaption>
</figure>

이어서 **Deploy model (manual) → 워크플로 실행**을 열고 두 값을 입력한다.

| 입력칸 | 넣는 값 |
|---|---|
| `image_tag` | 이미지 빌드가 끝난 로그에 나오는 `git-...` 태그 |
| `comparison_run_id` | 앞에서 세 모델을 비교한 MLflow Run ID |

<figure class="lab-shot" markdown="span">

[![Gitea 수동 배포 입력창. 빌드한 이미지 태그와 MLflow 모델 비교 Run ID를 지정한다](assets/model-serving-monitoring/gitea-deployment-inputs.jpg)](assets/model-serving-monitoring/gitea-deployment-inputs.jpg)

<figcaption>배포 입력창 · 위 칸은 실행할 코드 이미지, 아래 칸은 배포할 모델을 고르는 값이다.</figcaption>
</figure>

위 화면은 값을 입력한 상태다. 아래는 같은 이미지와 비교 Run을 사용해 앞서 완료한 **배포 #12**다.

<figure class="lab-shot" markdown="span">

[![Gitea Actions 배포 12번의 모델 확인, BentoML Pod 교체, API 예측 검사가 성공한 화면](assets/model-serving-monitoring/gitea-deployment-success.jpg)](assets/model-serving-monitoring/gitea-deployment-success.jpg)

<figcaption>Gitea Actions · 배포 #12. 모델 확인, 서빙 Pod 교체, API 예측 검사까지 성공했다.</figcaption>
</figure>

화면의 **Deliver image and update BentoML Pod**가 서버를 교체하는 단계다.
그다음 **Verify prediction API and record deployment**에서 시험 요청을 보내 예상한 예측이 나오는지 확인했다.
`Roll back on failure`는 실패가 없어 건너뛰었다.

**코드를 push하거나 학습을 완료한 것만으로 배포되지는 않는다.** 배포할 비교 Run을 사람이 선택한다.
Git에는 코드와 작업 정의를 저장하고, 데이터·모델 파일·인증값은 넣지 않았다.

```text
Pod 시작: 고정된 S3 경로 → 모델 직접 다운로드·해시 확인 → 메모리에 로딩
요청 처리: 8개 피처값 → 저장된 전처리 + predict() → 예측 결과
```

모델 파일은 MLflow 서버를 중계하지 않고 **Pod가 SeaweedFS에서 직접 다운로드**한다.
학습 때 사용한 문자열 정리와 원핫 인코더도 모델에 함께 저장해 요청마다 재사용한다.
현재 API는 8개 피처값을 직접 받으며, Feast 조회는 학습 데이터 준비에서 사용한다.

검증에서는 같은 worker가 모델을 재사용하는지, 예측이 학습 시 참조 결과와 일치하는지 확인했다.
배포 이력도 MLflow에 남겨 **데이터 버전 → 학습 → 비교·선택 → 배포 모델**을 따라갈 수 있게 했다.

### 배포된 API에 직접 요청하기

BentoML의 `/predict`를 펼치고 **Try it out**을 누른다.
직업·교육 등 8개 값을 JSON에 넣고 **Execute**를 눌렀다.
`collection_id`는 요청을 묶어 구분할 이름이다. 이번 요청에는 `blog-demo`를 넣었다.

<figure class="lab-shot" markdown="span">

[![BentoML predict 입력칸에 8개 피처와 blog-demo 요청 묶음을 입력한 화면](assets/model-serving-monitoring/bentoml-request.jpg)](assets/model-serving-monitoring/bentoml-request.jpg)

<figcaption>BentoML · 입력을 작성한 화면. Execute를 누르면 배포된 모델에 요청이 전달된다.</figcaption>
</figure>

<figure class="lab-shot" markdown="span">

[![BentoML predict의 실제 응답. HTTP 200과 소득 구간 예측, 모델 ID, 요청 기록 성공 여부가 표시된다](assets/model-serving-monitoring/bentoml-response.jpg)](assets/model-serving-monitoring/bentoml-response.jpg)

<figcaption>BentoML · 같은 입력에 대한 실제 응답. 이 예시는 소득이 5만 달러를 넘는 구간으로 예측했다.</figcaption>
</figure>

- **`200`**: API 호출이 정상 처리됐다는 뜻이다. 정답을 맞혔다는 뜻은 아니다.
- **`income: ">50K"`**: 모델이 예측한 소득 구간이다.
- **`model_id`, `training_run_id`**: 어떤 모델과 학습에서 나온 예측인지 MLflow에서 찾는 값이다.
- **`monitoring_logged: true`**: 이 요청과 예측을 SeaweedFS에 저장했다. 이렇게 모인 입력을 다음 Evidently 작업에서 비교한다.

## ④ 관측: Evidently로 무엇을 확인했나

모델을 배포한 다음에는 “요청이 성공했나?” 외에 **“지금 들어오는 데이터가 학습 때와 달라졌나?”**도 확인해야 한다.
Evidently는 Airflow 작업 안에서 학습 기준 데이터와 수집된 입력을 비교하고 보고서를 만든다.

```text
BentoML 요청·예측 기록 → SeaweedFS → Airflow 수동 분석 → MLflow 지표·보고서
```

| 비교 대상 | 이번 설정 |
|---|---|
| 기준 데이터 | 배포 모델을 학습한 Run의 학습 데이터 19,303행 |
| 현재 데이터 | 해당 모델·요청 묶음의 최근 최대 1,000행 또는 지정한 배치 파일 |
| 분석 범위 | 입력 8개 피처의 분포와 결측값 |
| 표본이 적을 때 | 200행 미만이면 분석 건너뜀 |

온라인 추론도 **모인 요청을 묶어서** 분포를 본다. 배치 입력은 준비된 파일을 바로 비교할 수 있다.

### 실제 확인한 결과

운영 트래픽 대신 공개 데이터로 모의 요청을 만들었다. 각 비교 대상은 1,000행이다.

| 사례 | 만든 입력 | 분석 결과 |
|---|---|---|
| 정상 분포 | 기준 데이터에서 표본 추출 | 드리프트 0 / 8개 |
| 분포 변화 | 직업·교육 등 6개 피처의 구성을 의도적으로 변경 | 드리프트 6 / 8개 |
| 결측값 | 직업 값 600개를 비운 별도 배치 | 직업 결측률 60% 확인 |

결측값 사례는 **배치 파일로 분석**했다. 현재 API는 누락·빈값을 거절하므로 이 요청들은 성공 요청 기록에 포함되지 않는다.
결측값 점검과 분포 변화 판정도 따로 봤다. 이번 결측 배치는 드리프트 0개였지만 직업 값의 누락을 확인할 수 있었다.

<figure class="lab-shot" markdown="span">

[![MLflow에 기록된 분포 변화 분석 결과. 기준 19,303행과 모의 요청 1,000행을 비교해 6개 피처의 변화를 감지했다.](assets/model-serving-monitoring/mlflow-drift-summary.jpg)](assets/model-serving-monitoring/mlflow-drift-summary.jpg)

<figcaption>MLflow · 변화 감지 피처 6개, 비율 0.75. 모델·학습 Run·데이터 버전도 함께 기록했다.</figcaption>
</figure>

보고서와 비교에 사용한 데이터·파일 해시도 보관했다.
범주형 Jensen–Shannon 거리 0.1, 전체 피처 중 변화 비율 0.5를 판정 기준으로 설정했다. 이는 이 실습의 설정이다.

<figure class="lab-shot" markdown="span">

[![MLflow에서 연 Evidently 요약 보고서의 직업 분포 차트. 학습 기준은 파랑, 현재 요청은 주황으로 표시된다](assets/model-serving-monitoring/evidently-distribution-report.jpg)](assets/model-serving-monitoring/evidently-distribution-report.jpg)

<figcaption>Evidently · 파랑은 학습 데이터, 주황은 모의 요청이다. 이 사례는 직업을 전부 Farming-fishing으로 바꿔 차이를 만들었다.</figcaption>
</figure>

Chrome에서 **Artifacts → monitoring → overview.html**을 열면 위 차트를 볼 수 있다.
예를 들어 Farming-fishing 비율은 학습 데이터에서 약 3.3%, 이 모의 요청에서는 100%다.

**드리프트는 오답이나 성능 저하를 뜻하지 않는다.** 정확도·F1은 정답 데이터가 있어야 평가할 수 있다.
이번 분석은 입력의 변화와 누락을 발견해 다음 실험의 근거를 남기는 데 목적을 뒀다.

## 막혔던 부분과 정리

| 문제 | 처리 |
|---|---|
| 학습과 API의 입력 처리 차이 | 전처리와 모델을 함께 저장하고 배포 후 참조 예측과 비교 |
| 온라인 요청이 아직 적음 | 최소 표본 수 미만이면 실패 대신 분석을 건너뜀 |
| 앱 내 브라우저에서 HTML 미리보기가 빈 화면 | Chrome에서 열거나, HTML을 다운로드해서 확인 |

여기까지는 API 응답과 입력의 변화를 확인했다.
다음에는 요청에 정답을 붙여 어떤 경우에 틀렸는지 확인하고, 데이터를 추가해 다시 학습할 차례다.
