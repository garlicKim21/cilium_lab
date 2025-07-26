# Cilium 환경에서 Prometheus & Grafana를 이용한 모니터링

이 문서는 Cilium 환경에서 Prometheus와 Grafana를 설치하고, 이를 활용하여 클러스터와 애플리케이션의 메트릭을 수집 및 시각화하는 방법을 정리합니다. 모든 내용은 제공된 스터디 자료에 기반합니다.

## 1. 모니터링과 관측 가능성(Observability)

시스템을 이해하는 두 가지 주요 접근 방식에는 모니터링과 관측 가능성이 있습니다.

### 모니터링(Monitoring)
사전에 정의된 메트릭(CPU, 메모리 등)을 추적하여 시스템의 상태를 감시하고 문제가 발생했는지 알려주는 것입니다. 주로 "시스템이 다운되었는가?"와 같은 **알려진 문제**를 감지하는 데 중점을 둡니다.

### 관측 가능성(Observability)
로그, 메트릭, 트레이스 등 시스템의 외부 출력 데이터를 통해 그 내부 상태를 이해하고, 예상치 못한 새로운 문제의 원인을 파악하는 능력입니다. "왜 특정 요청이 실패했는가?"와 같이 **알려지지 않은 문제**를 진단할 수 있게 해줍니다.

관측 가능성은 주로 **메트릭, 로그, 추적**이라는 세 가지 핵심 데이터(기둥)를 통해 이루어집니다.

### 관측 가능성의 3대 기둥 비교

| 비교 항목 | 메트릭 (Metrics) | 로그 (Logs) | 추적 (Tracing) |
|-----------|------------------|-------------|----------------|
| **정의** | 수치로 표현된 성능 데이터 | 시스템 이벤트 기록 | 요청의 시스템 통과 과정 추적 |
| **목적** | 시스템 성능 모니터링 및 알람 | 이벤트 분석 및 디버깅 | 서비스 간 호출 경로 및 병목 분석 |
| **도구** | Prometheus, Grafana | ELK Stack, Loki | Jaeger, Zipkin |

## 2. Prometheus & Grafana 설치

실습 환경에서는 사전 구성된 YAML 파일을 통해 Prometheus와 Grafana 스택을 간편하게 배포합니다.

### Prometheus & Grafana 스택 배포

`monitoring-example.yaml` 파일을 적용하여 Prometheus, Grafana 및 관련 설정을 한 번에 배포합니다. 이 설정에는 Cilium 및 Hubble 대시보드가 Grafana에 자동으로 주입되는 내용이 포함됩니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/kubernetes/addons/prometheus/monitoring-example.yaml
```

### Cilium 및 Hubble 메트릭 활성화

Cilium 에이전트, Operator, Hubble이 메트릭을 노출하도록 Helm 설정을 통해 활성화해야 합니다. 실습 환경에서는 이미 설정이 완료되어 있습니다.

- `prometheus.enabled=true`: Cilium 에이전트의 메트릭을 활성화합니다.
- `operator.prometheus.enabled=true`: Cilium Operator의 메트릭을 활성화합니다.
- `hubble.metrics.enabled`: Hubble의 메트릭 목록을 활성화합니다.

### 외부 접속을 위한 NodePort 설정

배포된 Prometheus와 Grafana 서비스는 기본적으로 `ClusterIP` 타입이므로, 외부에서 웹 UI에 접속하기 위해 NodePort로 변경합니다.

```bash
# Prometheus 서비스 NodePort로 변경
kubectl patch svc -n cilium-monitoring prometheus -p '{"spec": {"type": "NodePort", "ports": [{"port": 9090, "nodePort": 30001}]}}'

# Grafana 서비스 NodePort로 변경
kubectl patch svc -n cilium-monitoring grafana -p '{"spec": {"type": "NodePort", "ports": [{"port": 3000, "nodePort": 30002}]}}'
```

설정 후 노드 IP와 포트(예: `http://192.168.10.100:30001` for Prometheus)를 통해 접속할 수 있습니다.

## 3. Prometheus & Grafana 사용법

### Prometheus UI

#### Status → Targets
Prometheus가 현재 메트릭을 수집하고 있는 대상(endpoint)들의 상태를 확인할 수 있습니다.

#### Status → Configuration
현재 적용된 Prometheus의 설정 파일을 조회할 수 있습니다. ServiceMonitor를 통해 어떻게 대상을 동적으로 발견하는지 등을 확인할 수 있습니다.

#### Graph
PromQL(Prometheus Query Language)을 사용해 메트릭을 직접 쿼리하고 그래프로 시각화할 수 있습니다.

### Grafana UI

#### Data Sources
Grafana가 데이터를 가져올 소스를 설정합니다. 배포된 스택에는 Prometheus가 `http://prometheus:9090` 주소로 이미 등록되어 있습니다.

#### Dashboards
Prometheus로 수집한 데이터를 시각화하는 공간입니다. `monitoring-example.yaml`을 통해 Cilium Metrics, Cilium Operator, Hubble, Hubble L7 HTTP Metrics 등의 대시보드가 미리 생성되어 있습니다.

## 4. PromQL 쿼리 예제 분석

Grafana의 "Cilium Metrics" 대시보드에 있는 `map ops` 패널의 쿼리를 통해 PromQL의 동작을 이해할 수 있습니다.

### 예제 쿼리 분석

```promql
topk(5, avg(rate(cilium_bpf_map_ops_total{k8s_app="cilium", pod=~"$pod"}[5m])) by (pod, map_name, operation))
```

**단계별 분석:**

1. **`cilium_bpf_map_ops_total{...}`**: cilium 앱에 속한 파드들의 `cilium_bpf_map_ops_total`이라는 Counter 타입 메트릭을 선택합니다.

2. **`rate(...[5m])`**: 위 메트릭의 최근 5분간 초당 평균 증가율을 계산합니다.

3. **`avg(...) by (pod, map_name, operation)`**: 계산된 증가율을 pod, map_name, operation 레이블을 기준으로 그룹화하여 평균을 냅니다.

4. **`topk(5, ...)`**: 그룹화된 결과 중 가장 값이 큰 상위 5개를 선택하여 보여줍니다.

## 5. 커스텀 Grafana 대시보드 만들기

Grafana에서는 PromQL을 사용하여 다양한 시각화 패널로 구성된 나만의 대시보드를 만들 수 있습니다.

### 패널 타입별 활용법

| 패널 타입 | 설명 | 예시 PromQL 쿼리 |
|-----------|------|-------------------|
| **Gauge** | 단일 메트릭이 임계값 대비 어느 정도인지를 보여줍니다. | `1 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) by (instance))` (노드별 CPU 사용률) |
| **Bar chart** | 범주형 데이터를 막대그래프로 보여줍니다. | `count(kube_deployment_status_replicas_available) by (namespace)` (네임스페이스별 디플로이먼트 개수) |
| **Stat** | 중요한 단일 수치를 크게 표시합니다. | `kube_deployment_spec_replicas{deployment="nginx"}` (Nginx 파드 개수) |
| **Time series** | 시간에 따른 데이터 변화를 그래프로 보여줍니다. | `sum(rate(node_cpu_seconds_total[5m])) by (instance)` (노드별 5분간 CPU 사용 변화율) |
| **Table** | 데이터를 테이블 형태로 보여줍니다. | `node_os_info` (노드 OS 정보) |

## 6. Grafana Alerting

Grafana를 사용하여 특정 조건이 충족되었을 때 알림을 보낼 수 있습니다.

### 알림 설정 구성 요소

#### Contact points
슬랙(Slack), 이메일 등 알림을 수신할 채널을 설정합니다.

#### Notification policy
어떤 알림을 어떤 Contact point로 보낼지 정책을 정의합니다.

#### Alert rule
특정 PromQL 쿼리 결과가 임계값을 넘는 등 알림을 발생시킬 조건을 생성합니다.

### 알림 설정 순서

1. **Contact points 설정**: 알림을 받을 채널(이메일, 슬랙 등) 구성
2. **Notification policy 정의**: 알림 라우팅 규칙 설정
3. **Alert rule 생성**: 알림 발생 조건 및 임계값 설정
4. **테스트 및 모니터링**: 설정된 알림이 정상적으로 동작하는지 확인

## 결론

Prometheus와 Grafana를 통한 모니터링 시스템은 Cilium 환경에서 네트워크 성능과 보안을 효과적으로 관찰할 수 있게 해줍니다. PromQL을 활용한 메트릭 쿼리와 Grafana의 다양한 시각화 옵션을 통해 시스템의 상태를 실시간으로 파악하고, 알림 기능을 통해 문제 상황에 신속하게 대응할 수 있습니다. 