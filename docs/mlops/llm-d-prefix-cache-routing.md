---
title: "MLOps 실습 4 · llm-d의 Prefix Cache 라우팅은 언제 빨라질까?"
---

앞선 실습에서는 모델을 학습하고 배포했다. 이번에는 **이미 떠 있는 vLLM 복제본 네 개 중 어디로 요청을 보내느냐**에 따라 응답 속도가 얼마나 달라지는지 살펴봤다.

여러 긴 프롬프트가 고르게 반복됐을 때, 캐시를 고려한 라우팅은 Round Robin보다 초당 완료 요청 수를 **10.7%** 늘리고 전체 응답 시간 p90을 **17.4%** 줄였다. 같은 실험에서 **p99는 22.8% 늘었다.** 대부분의 요청이 빨라져도 가장 느린 쪽의 요청까지 빨라지는 것은 아니었다.

## 트랜스포머에서 KV 캐시는 어디에 있나

Qwen처럼 텍스트를 이어 쓰는 모델은 **디코더 블록**을 여러 층 쌓아 다음 토큰을 예측한다. 각 층의 셀프 어텐션은 현재 토큰이 앞의 토큰을 얼마나 참고할지 계산한다. 아래 그림은 정규화와 잔차 연결을 생략한 구조다.

<figure class="routing-diagram">
<div class="routing-flow"><span>입력 토큰</span><b aria-hidden="true">→</b><span>임베딩</span><b aria-hidden="true">→</b><span>디코더 블록 × 여러 층</span><b aria-hidden="true">→</b><span>다음 토큰 예측</span></div>
<div class="routing-lane">
<strong>디코더 블록 한 층</strong>
<div class="routing-flow"><span>현재 토큰의 Q</span><b aria-hidden="true">+</b><span>캐시의 K·V + 현재 K·V</span><b aria-hidden="true">→</b><span>셀프 어텐션</span><b aria-hidden="true">→</b><span>피드포워드 층</span></div>
<div class="routing-flow"><span>현재 토큰의 K·V</span><b aria-hidden="true">→</b><span>이 층의 KV 캐시에 추가</span></div>
</div>
<figcaption>응답 생성 중 각 층은 앞 토큰의 K·V를 읽고 현재 토큰의 K·V를 저장한다.</figcaption>
</figure>

**KV 캐시**는 어텐션이 이미 처리한 토큰의 Key와 Value 벡터를 층마다 저장한 것이다. 새 토큰을 만들 때 그 토큰의 Query를 이전 Key들과 비교하고, 그 결과에 따라 이전 Value들을 섞는다. 저장해 둔 K·V를 읽으면 앞 토큰의 K·V를 다시 계산하는 일을 줄일 수 있다.

| 단계 | 모델이 하는 일 | 캐시 사용 |
|---|---|---|
| Prefill | 프롬프트를 처리해 응답 생성을 준비 | 입력 토큰의 K·V 저장 |
| Decode | 응답을 한 토큰씩 생성 | 앞 토큰의 K·V를 읽고 새 K·V 추가 |
| 다음 요청의 Prefill | 앞부분이 같은 새 프롬프트 처리 | APC가 재사용 가능한 공통 토큰 블록의 계산을 건너뜀 |

앞의 두 행은 한 요청 안에서 일어나는 일이다. 마지막 행은 vLLM의 **Automatic Prefix Caching(APC)**으로 여러 요청 사이에 같은 입력 앞부분을 재사용하는 경우다. 이 재사용으로 줄이는 비용은 새 요청의 prefill에 해당한다. [vLLM APC 문서](https://docs.vllm.ai/en/stable/features/automatic_prefix_caching/)

## llm-d는 요청을 어디로 보낼까

llm-d는 Kubernetes에서 여러 LLM 추론 서버를 운영하기 위한 오픈소스 프로젝트다. 이번 실험에서는 그중 **Endpoint Picker(EPP)**를 이용한 요청 배치를 비교했다. EPP는 설정에 따라 입력 앞부분의 캐시 재사용 가능성과 복제본의 대기열·부하를 점수에 반영한다. 요청을 받은 복제본에서 실제 모델 실행과 캐시 관리는 vLLM이 맡는다. [llm-d 라우팅 문서](https://github.com/llm-d/llm-d/blob/main/docs/architecture/advanced/kv-management/prefix-cache-aware-routing.md)

제품 매뉴얼 챗봇에 아래 두 질문이 들어온다고 하자.

```text
[시스템 지시 + 제품 매뉴얼 A] + "초기 설정은 어떻게 해?"
[시스템 지시 + 제품 매뉴얼 A] + "설정을 초기화하려면?"
```

질문은 달라도 프롬프트 앞부분의 토큰은 같다. 이 실험에서 캐시는 복제본마다 따로 있으므로 같은 문서를 붙여도 어느 복제본으로 가느냐에 따라 재사용 여부가 달라진다.

<figure class="routing-diagram">
<div class="routing-lane">
<strong>Round Robin</strong>
<div class="routing-flow"><span>요청 1 · 매뉴얼 A</span><b aria-hidden="true">→</b><span>vLLM 0</span><b aria-hidden="true">→</b><span>A 캐시 생성</span></div>
<div class="routing-flow"><span>요청 2 · 매뉴얼 A</span><b aria-hidden="true">→</b><span>vLLM 1</span><b aria-hidden="true">→</b><span>A 다시 계산</span></div>
</div>
<div class="routing-lane">
<strong>llm-d · 캐시를 고려한 라우팅</strong>
<div class="routing-flow"><span>요청 1 · 매뉴얼 A</span><b aria-hidden="true">→</b><span>vLLM 0</span><b aria-hidden="true">→</b><span>A 캐시 생성</span></div>
<div class="routing-flow"><span>요청 2 · 매뉴얼 A</span><b aria-hidden="true">→</b><span>vLLM 0</span><b aria-hidden="true">→</b><span>A 캐시 재사용 가능</span></div>
</div>
<figcaption>첫 요청은 빈 캐시에서 시작한다. 두 번째 요청이 올 때 vLLM 0에는 매뉴얼 A의 캐시와 요청을 처리할 여유가 있다.</figcaption>
</figure>

두 번째 요청은 첫 요청이 처리된 뒤 도착한다. 실제 라우팅에서는 캐시가 있는 복제본도 대기열이 길면 우선순위가 낮아질 수 있다.

## 어떻게 비교했나

| 항목 | 실험 조건 |
|---|---|
| 모델과 실행 환경 | Qwen2.5-7B-Instruct, vLLM, 최대 입력·출력 합계 4,096토큰 설정 |
| GPU와 복제본 | H100 80GB 4장, 복제본당 1장 |
| 고정 조건 | 동일한 워크로드와 동시성, 양쪽 모두 vLLM APC 활성화 |
| 비교 대상 | Round Robin(RR)과 llm-d의 Prefix Cache를 고려한 라우팅 |

방식별로 같은 워크로드를 5회씩 실행했다. 각 지표에서 최댓값과 최솟값을 하나씩 빼고 나머지 3회 평균을 냈다. 처리량은 **성공 요청 수 ÷ 첫 요청 시작부터 마지막 요청 완료까지 걸린 시간**으로 계산했다. 서버 기동과 모델 로딩은 이 시간에 포함하지 않았다.

**TTFT**는 첫 토큰까지, **E2E**는 응답이 끝날 때까지 걸린 시간이다. 지연의 p90과 p99는 각각 요청의 약 90%와 99%가 그 시간 안에 해당 지점에 도달했다는 뜻이다.

## 네 가지 요청 패턴의 결과

<figure class="routing-chart">
<figcaption>Round Robin 대비 변화율. 가운데가 0%이며, 처리량은 오른쪽, 지연은 왼쪽으로 갈수록 좋다. 세 그래프의 눈금은 모두 같다.</figcaption>
<div class="routing-panels">
<section aria-label="초당 완료 요청 변화율"><h3>초당 완료 요청</h3>
<div class="routing-axis" aria-hidden="true"><span>−25%</span><span>0</span><span>+25%</span></div>
<div class="routing-label"><span>여러 문서 균등</span><span>+10.7% · 개선</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar better" style="left:50.000%;width:21.349%"></span></div>
<div class="routing-label"><span>멀티턴 세션</span><span>약 0% · 거의 동일</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar worse" style="left:49.899%;width:0.101%"></span></div>
<div class="routing-label"><span>단일 문서</span><span>-14.8% · 악화</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar worse" style="left:20.478%;width:29.522%"></span></div>
<div class="routing-label"><span>인기 문서 쏠림</span><span>-2.7% · 악화</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar worse" style="left:44.564%;width:5.436%"></span></div>
</section>
<section aria-label="전체 응답 p90 변화율"><h3>전체 응답 p90</h3>
<div class="routing-axis" aria-hidden="true"><span>−25%</span><span>0</span><span>+25%</span></div>
<div class="routing-label"><span>여러 문서 균등</span><span>-17.4% · 개선</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar better" style="left:15.215%;width:34.785%"></span></div>
<div class="routing-label"><span>멀티턴 세션</span><span>-17.2% · 개선</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar better" style="left:15.669%;width:34.331%"></span></div>
<div class="routing-label"><span>단일 문서</span><span>+4.8% · 악화</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar worse" style="left:50.000%;width:9.594%"></span></div>
<div class="routing-label"><span>인기 문서 쏠림</span><span>+4.6% · 악화</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar worse" style="left:50.000%;width:9.142%"></span></div>
</section>
<section aria-label="전체 응답 p99 변화율"><h3>전체 응답 p99</h3>
<div class="routing-axis" aria-hidden="true"><span>−25%</span><span>0</span><span>+25%</span></div>
<div class="routing-label"><span>여러 문서 균등</span><span>+22.8% · 악화</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar worse" style="left:50.000%;width:45.625%"></span></div>
<div class="routing-label"><span>멀티턴 세션</span><span>-15.6% · 개선</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar better" style="left:18.768%;width:31.232%"></span></div>
<div class="routing-label"><span>단일 문서</span><span>+5.9% · 악화</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar worse" style="left:50.000%;width:11.893%"></span></div>
<div class="routing-label"><span>인기 문서 쏠림</span><span>+0.8% · 악화</span></div>
<div class="routing-track" aria-hidden="true"><span class="routing-bar worse" style="left:50.000%;width:1.553%"></span></div>
</section>
</div>
</figure>

표는 **Round Robin → 캐시를 고려한 라우팅** 순서다. 초당 완료 요청 수는 클수록, 지연 시간은 작을수록 좋다.

| 요청 패턴 | 초당 완료 요청 | TTFT p90 (ms) | E2E p90 (ms) | E2E p99 (ms) |
|---|---:|---:|---:|---:|
| 여러 문서가 고르게 반복 | 92.09 → **101.92** | 53.81 → **47.34** | 110.68 → **91.43** | **135.67** → 166.62 |
| 사용자별 멀티턴 세션 | 39.62 → 39.60 | 42.15 → **28.41** | 85.87 → **71.13** | 102.78 → **86.73** |
| 단일 인기 문서 반복 | **111.24** → 94.82 | **53.10** → 56.98 | **94.23** → 98.75 | **100.90** → 106.90 |
| 소수 인기 문서에 집중 | **114.05** → 110.95 | **29.94** → 32.99 | **72.85** → 76.18 | **86.31** → 86.98 |

### 여러 문서가 고르게 반복된 경우

제품 Q&A에서 설치, 초기화, 오류 코드처럼 서로 다른 매뉴얼이 비슷한 빈도로 조회되는 상황이다. Round Robin에서는 같은 매뉴얼이 여러 복제본으로 흩어진다. 캐시를 고려하면 이미 해당 매뉴얼을 계산한 복제본을 다시 쓸 기회가 생기고, 서로 다른 매뉴얼의 요청도 여러 복제본에 나뉠 수 있다.

이 경우 초당 완료 요청이 **92.09건에서 101.92건**으로 늘고 E2E p90은 **110.68ms에서 91.43ms**로 줄었다. 반면 E2E p99는 **135.67ms에서 166.62ms**로 늘었다. 응답 시간 목표가 p90 기준인지 p99 기준인지에 따라 같은 결과를 보고도 라우팅 선택이 달라질 수 있다.

### 사용자별 멀티턴 세션

한 사용자가 “설치 방법을 알려줘”라고 한 뒤 “두 번째 단계만 더 자세히 설명해 줘”라고 묻는 상황이다. 다음 요청에 이전 대화가 포함되면 같은 세션의 앞부분을 다시 쓸 수 있다. 여러 사용자의 대화가 동시에 진행되더라도 각 세션에 반복되는 입력은 다르다.

초당 완료 요청은 **39.62건과 39.60건**으로 거의 같았다. 첫 토큰 p90은 **32.6%**, 전체 응답 p99는 **15.6%** 줄었다. 이 경우 사용자가 답변을 보기 시작하기까지의 대기 시간과 느린 응답의 지연이 줄었다는 점이 성과다.

### 단일 인기 문서가 반복된 경우

모든 질문에 같은 제품 매뉴얼이 붙는 상황이다. 초반에는 Round Robin이 같은 매뉴얼을 여러 복제본에서 계산한다. 하지만 캐시가 유지된다면 워밍업 이후에는 어느 복제본에 보내도 그 매뉴얼을 재사용할 수 있다. 이때 일부 복제본을 우선하면 요청을 고르게 분산하는 이점이 줄어들 수 있다.

이번에는 Round Robin이 표의 네 지표에서 모두 앞섰다. 캐시를 고려한 방식에서는 초당 완료 요청이 **14.8% 줄었고**, E2E p99는 **5.9% 늘었다**.

### 소수 인기 문서에 요청이 몰린 경우

배송, 환불, 계정 등 여러 문서가 있지만 환불 정책만 유난히 자주 조회되는 FAQ를 생각하면 된다. 문서별로 캐시를 재사용할 기회는 있어도, 인기 문서 요청이 같은 복제본에 집중되면 줄이 길어질 수 있다. 대기 시간이 길어지면 prefill에서 아낀 시간을 상쇄할 수 있다.

이번 설정에서 초당 완료 요청은 **2.7% 줄고**, E2E p90은 **4.6% 늘었다**. 이 결과로 요청 쏠림이 원인이었다고 확정할 수는 없다. 다음 비교에서는 복제본별 요청 수와 큐 길이를 기록해, 인기 문서 요청이 실제로 한쪽에 몰렸는지 볼 수 있다.

## 운영 트래픽에서 확인할 것

비교의 출발점은 **같은 긴 입력이 얼마나 자주 반복되고, 그 요청이 얼마나 고르게 나뉘는가**다. RAG 검색 결과의 문서 순서가 매번 바뀌거나 요청마다 다른 값이 프롬프트 앞에 들어가면, 같은 문서를 쓰더라도 연속해서 일치하는 토큰 구간이 짧아질 수 있다.

이번 결과는 GPU 4장, 모델 하나, 합성 요청으로 측정한 값이다. 처리량 개선은 이 조건에서 같은 시간에 더 많은 요청을 완료했다는 뜻이다. p99 악화의 원인으로 큐 대기나 배칭을 생각해 볼 수 있지만, 이 결과만으로 원인을 특정할 수는 없다. 운영에서 라우팅을 비교할 때는 **복제본별 요청 수·대기열·캐시 적중률**을 함께 보고, 처리량뿐 아니라 **p99와 목표 응답 시간 충족률**도 확인해야 한다.

다음 실험에서는 인기 문서의 요청 비중을 단계적으로 바꾸고, 캐시 재사용과 큐 길이에 부여하는 가중치를 조정해 보고 싶다. 같은 워크로드에 **현재 처리 중인 요청이 적은 복제본을 우선하는 방식(Least Request)**도 추가하면, 요청 패턴에 따라 캐시를 우선할 때와 부하를 우선할 때의 차이를 확인할 수 있다.

## 결론 {#conclusion}

이번 실험에서 llm-d의 효과는 반복되는 프롬프트의 종류와 요청 분포에 따라 달랐다. 여러 긴 프롬프트가 고르게 반복된 경우에는 **처리량이 10.7% 늘고 p90이 17.4% 줄었지만, p99는 22.8% 늘었다.** 멀티턴 세션은 처리량 변화 없이 p99가 줄었고, 한 문서만 반복되거나 몇몇 문서에 요청이 몰린 경우에는 Round Robin이 앞섰다.

서비스에 적용할 때는 실제 요청 분포를 재현한 뒤, **목표 응답 시간을 지키면서 초당 몇 건을 처리할 수 있는지** 두 방식에서 비교하는 게 좋다. p99가 목표를 넘는다면 복제본별 대기열과 캐시 적중률을 함께 확인하고, 캐시 재사용과 부하에 주는 가중치를 조정해 다시 측정한다.
