# [Cilium Study] 2주차 - 2. Prometheus와 Grafana로 시스템 들여다보기

지난 [Hubble 포스팅](https://tech-recipe.tistory.com/38)에서는 마치 어두운 방에 불을 켜듯, 클러스터 내부의 네트워크 흐름을 눈으로 직접 확인하는 경험을 했습니다. 정말 속이 다 시원해지는 느낌이었죠. 하지만 '보는 것'만으로는 충분하지 않을 때가 있습니다. "CPU 사용량이 평소보다 높은 것 같은데, 얼마나 높지?", "어제 새벽에 잠깐 장애가 있었던 것 같은데, 정확히 어떤 일이 있었을까?" 와 같은 질문에 답하려면, 시스템의 상태를 숫자로 측정하고 기록해두는 과정이 반드시 필요합니다.

바로 여기서 **모니터링(Monitoring)** 의 세계가 시작됩니다. 이번 포스팅에서는 Cilium 환경에서 모니터링의 대명사, **Prometheus(프로메테우스)** 와 그의 단짝 **Grafana(그라파나)** 를 설치하고, 이를 통해 클러스터의 건강 상태를 진단하고 시각화하는 여정을 떠나보겠습니다.

---

## 1. 모니터링 vs 관측 가능성, 그 미묘한 차이

개발자라면 '모니터링'과 '관측 가능성(Observability)'이라는 단어를 정말 많이 들어보셨을 겁니다. 저도 처음에는 두 개가 그냥 비슷한 말인 줄 알았습니다. 하지만 이 둘은 시스템을 이해하는 서로 다른 접근 방식입니다.

* **모니터링 (Monitoring):** "우리 서버 CPU 사용률이 90%를 넘었어!" 처럼, 우리가 **미리 알고 있는** 문제(Known-Unknowns)를 감시하고 알람을 울리는 것입니다. 마치 건강검진에서 혈압이나 혈당 수치를 재는 것과 비슷하죠.
* **관측 가능성 (Observability):** "CPU 사용률이 왜 갑자기 90%를 넘었을까? 어떤 요청 때문에?" 처럼, 전혀 **예상하지 못했던** 문제(Unknown-Unknowns)의 근본 원인을 파악하는 능력입니다. 증상만 보는 것이 아니라, 로그, 메트릭, 트레이스 등 다양한 단서를 조합해 병의 원인을 진단하는 명의 같다고 할까요?

이 관측 가능성을 떠받치는 세 개의 기둥이 바로 **메트릭, 로그, 추적(Trace)** 입니다.

| 비교 항목 | 메트릭 (Metrics) | 로그 (Logs) | 추적 (Tracing) |
| :-------- | :--------------------- | :----------------- | :----------------------------- |
| **정의** | 숫자로 표현된 성능 데이터 | 시스템 이벤트 기록 | 요청의 시스템 통과 과정 추적 |
| **목적** | 시스템 성능 모니터링 및 알람 | 이벤트 분석 및 디버깅 | 서비스 간 호출 경로 및 병목 분석 |
| **대표 도구** | **Prometheus**, Grafana | ELK Stack, Loki | Jaeger, Zipkin |

오늘 우리가 다룰 Prometheus와 Grafana는 이 중 **메트릭**을 수집하고 시각화하는 데 가장 강력한 도구입니다.

---

## 2. Prometheus & Grafana 설치하기

다행히도, Cilium 커뮤니티에서 미리 만들어 둔 '모니터링 종합 선물 세트'가 있습니다. 우리는 마법의 주문(`kubectl apply`) 한 줄로 Prometheus와 Grafana 스택을 손쉽게 배포할 수 있습니다.

### Prometheus & Grafana 스택 배포
아래 `monitoring-example.yaml` 파일에는 Prometheus와 Grafana는 물론, Cilium과 Hubble 대시보드까지 Grafana에 자동으로 쏙 넣어주는 설정이 모두 포함되어 있습니다. 정말 편리하죠.

```bash
kubectl apply -f [https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/kubernetes/addons/prometheus/monitoring-example.yaml](https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/kubernetes/addons/prometheus/monitoring-example.yaml)
```

### Cilium 및 Hubble 메트릭 활성화
Prometheus가 데이터를 수집하려면, Cilium과 Hubble이 "우리 데이터 여기 있어!" 하고 메트릭을 외부에 노출해줘야 합니다. 이 설정은 지난번 Hubble을 설치할 때 이미 Helm 명령어를 통해 활성화했습니다. 기억나시죠?

* `prometheus.enabled=true`: Cilium 에이전트 메트릭 활성화
* `operator.prometheus.enabled=true`: Cilium Operator 메트릭 활성화
* `hubble.metrics.enabled`: Hubble 메트릭 활성화

### 외부에서 접속할 수 있도록 문 열어주기
기본적으로 설치된 Prometheus와 Grafana 서비스는 클러스터 안에서만 접근 가능한 `ClusterIP` 타입입니다. 우리가 웹 브라우저로 UI를 보려면 외부에서 접근할 수 있는 `NodePort`로 변경해줘야 합니다.

```bash
# Prometheus 서비스 타입을 NodePort로 변경
kubectl patch svc -n cilium-monitoring prometheus -p '{"spec": {"type": "NodePort", "ports": [{"port": 9090, "nodePort": 30001}]}}'

# Grafana 서비스 타입을 NodePort로 변경
kubectl patch svc -n cilium-monitoring grafana -p '{"spec": {"type": "NodePort", "ports": [{"port": 3000, "nodePort": 30002}]}}'
```

이제 `http://<노드IP>:30001` 로는 Prometheus에, `http://<노드IP>:30002` 로는 Grafana에 접속할 수 있게 되었습니다!

---

## 3. Prometheus & Grafana 둘러보기

### Prometheus UI: 데이터 창고지기

Prometheus는 데이터를 수집하고 저장하는 창고와 같습니다. UI는 화려하진 않지만, 아주 중요한 정보들을 담고 있습니다.

* **Status → Targets:** Prometheus가 현재 데이터를 긁어오고 있는 대상(endpoint)들의 목록과 건강 상태를 보여줍니다. 마치 출석부 같죠. 여기서 `UP` 상태가 아니라면 뭔가 문제가 생긴 겁니다.
* **Status → Configuration:** 현재 Prometheus에 적용된 설정 파일을 보여줍니다. ServiceMonitor 같은 기능을 통해 어떻게 동적으로 서비스들을 찾아내는지 엿볼 수 있습니다.
* **Graph:** PromQL(Prometheus Query Language)이라는 강력한 쿼리 언어로 데이터를 직접 조회하고 간단한 그래프로 볼 수 있는 곳입니다.

### Grafana UI: 데이터 시각화의 마법사

Grafana는 Prometheus라는 창고에서 데이터를 가져와 멋지고 이해하기 쉬운 대시보드로 만들어주는 시각화 도구입니다.

* **Data Sources:** Grafana가 데이터를 가져올 창고를 설정하는 곳입니다. 우리는 이미 `monitoring-example.yaml`을 통해 Prometheus가 등록되어 있는 것을 확인할 수 있습니다.
* **Dashboards:** 데이터 시각화의 꽃, 대시보드들이 모여있는 곳입니다. Cilium Metrics, Hubble 등 이미 만들어진 멋진 대시보드들을 바로 사용할 수 있습니다.

---

## 4. PromQL 쿼리 살짝 맛보기

Grafana 대시보드를 보면 복잡해 보이는 쿼리들이 많은데요, 그중 하나를 뜯어보면서 PromQL과 친해져 보겠습니다. "Cilium Metrics" 대시보드의 `map ops` 패널 쿼리입니다.

```promql
topk(5, avg(rate(cilium_bpf_map_ops_total{k8s_app="cilium", pod=~"$pod"}[5m])) by (pod, map_name, operation))
```

처음 보면 외계어 같지만, 차근차근 분해해보면 별거 아닙니다.

1.  `cilium_bpf_map_ops_total{...}`: "일단 `cilium_bpf_map_ops_total` 이라는 이름의 메트릭 데이터 뭉치를 가져와!" 라는 뜻입니다.
2.  `rate(...[5m])`: 그 데이터가 계속 누적되는 값이니, "최근 5분 동안 초당 평균 얼마나 증가했는지 변화율을 계산해줘." 라는 의미입니다.
3.  `avg(...) by (pod, map_name, operation)`: 계산된 변화율을 "파드 이름, 맵 이름, 작업 종류별로 그룹지어서 평균을 내줘." 라고 요청하는 겁니다.
4.  `topk(5, ...)`: 마지막으로 "그 결과들 중에서 값이 제일 높은 놈들로 5개만 보여줘!" 하고 마무리합니다.

어떤가요? 이렇게 보니 영어를 해석하는 것 같지 않나요? PromQL은 이런 식으로 데이터를 조합하고 가공하는 데 아주 강력한 능력을 보여줍니다.

---

## 5. 나만의 Grafana 대시보드 만들기

기존 대시보드도 훌륭하지만, 내가 꼭 보고 싶은 정보만 모아서 나만의 관제 센터를 만들고 싶을 때가 있죠. Grafana에서는 아주 쉽게 커스텀 대시보드를 만들 수 있습니다.

| 패널 타입 | 설명 | 예시 PromQL 쿼리 |
| :---------- | :--------------------------------------------- | :--------------------------------------------------------------------------------------------------- |
| **Gauge** | 단일 값이 임계치에 비해 어느 정도인지 보여줍니다. | `1 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) by (instance))` (노드별 CPU 사용률) |
| **Bar chart** | 항목별 데이터를 막대그래프로 비교할 때 좋습니다. | `count(kube_deployment_status_replicas_available) by (namespace)` (네임스페이스별 디플로이먼트 개수) |
| **Stat** | 중요한 숫자 하나를 크고 아름답게 보여줍니다. | `kube_deployment_spec_replicas{deployment="nginx"}` (Nginx 파드 개수) |
| **Time series**| 시간에 따른 데이터 변화를 추적할 때 사용합니다. | `sum(rate(node_cpu_seconds_total[5m])) by (instance)` (노드별 5분간 CPU 사용 변화율) |
| **Table** | 여러 정보를 표 형태로 깔끔하게 정리해 보여줍니다. | `node_os_info` (노드 OS 정보) |

---

## 6. Grafana Alerting: 문제가 생기면 알려줘!

모니터링의 핵심은 문제가 생겼을 때 재빨리 알아차리는 것입니다. Grafana는 특정 조건이 만족되면 우리에게 알려주는 똑똑한 알림(Alerting) 기능을 제공합니다.

설정 순서는 간단합니다.

1.  **Contact points 설정:** "어디로 알려줄까?" (이메일, 슬랙 등)
2.  **Notification policy 정의:** "어떤 종류의 알림을 누구에게 보낼까?"
3.  **Alert rule 생성:** "CPU 사용률이 5분 동안 90%를 넘으면 알려줘!" 와 같이 알림을 울릴 정확한 조건을 정의합니다.

이렇게 설정해두면, 밤에 편히 잠을 자더라도 시스템에 문제가 생겼을 때 바로 알림을 받을 수 있게 됩니다.

## 마무리하며

Prometheus와 Grafana를 통해 우리는 Cilium 환경의 네트워크 성능과 보안 상태를 효과적으로 측정하고 관찰할 수 있게 되었습니다. Hubble로 '보고', Prometheus로 '측정'하고, Grafana로 '시각화'하며 '알림'까지 받는 이 조합은, 우리가 시스템을 훨씬 깊이 있게 이해하고 안정적으로 운영할 수 있는 강력한 무기가 되어줄 겁니다.

단순히 CNI를 설치하는 것을 넘어, 그 내부를 들여다보고 제어하는 경험을 통해 쿠버네티스 네트워킹에 대한 자신감을 한층 더 얻을 수 있었던 시간이었습니다.

이것으로 '[Cilium Study] 2주차' 포스팅을 마무리하겠습니다. 감사합니다.

