# Cilium & Hubble을 이용한 네트워크 관측(Observability)

이 문서는 Cilium Hubble을 사용하여 쿠버네티스 클러스터의 네트워크를 관찰하고 정책을 적용하는 방법을 정리합니다. 모든 내용은 제공된 스터디 자료에 기반합니다.

## 1. Hubble 소개

Hubble은 Cilium과 eBPF를 기반으로 구축된 분산 네트워킹 및 보안 관측 플랫폼입니다. eBPF를 활용하여 낮은 오버헤드로 서비스 통신, 동작, 네트워킹 인프라에 대한 깊은 가시성을 제공합니다.

### Hubble의 주요 기능

#### 서비스 의존성 및 통신 맵 (Service dependencies & communication map)
- 서비스 간의 통신 여부, 빈도, 의존성 그래프를 시각화
- HTTP 호출이나 Kafka 토픽 같은 L7 트래픽을 확인

#### 네트워크 모니터링 및 알림 (Network monitoring & alerting)
- 네트워크 통신의 실패 여부와 그 원인(DNS, TCP, HTTP 등)을 파악
- 최근 5분간 DNS 확인에 실패했거나 TCP 연결이 중단된 서비스를 찾을 수 있음

#### 애플리케이션 모니터링 (Application monitoring)
- 특정 서비스의 HTTP 4xx/5xx 응답 코드 비율 측정
- 95/99번째 백분위수 지연 시간을 측정

#### 보안 관측 (Security observability)
- 네트워크 정책에 의해 차단된 연결을 확인
- 클러스터 외부에서 접근한 서비스나 특정 DNS 이름을 확인한 서비스를 추적

### Hubble 구성 요소

#### Hubble API
- 기본적으로 각 노드의 Cilium 에이전트 내에서 로컬로 관찰된 트래픽에 대한 인사이트를 제공

#### Hubble CLI
- 로컬 유닉스 도메인 소켓을 통해 Hubble API를 쿼리하는 데 사용
- Cilium 에이전트 파드에 기본적으로 설치

#### Hubble Relay
- 클러스터 전체 또는 여러 클러스터에 대한 네트워크 가시성을 제공하는 서비스

#### Hubble UI
- Hubble Relay를 통해 수집된 데이터를 사용자 친화적인 서비스 맵과 데이터 흐름 필터링 기능이 있는 웹 인터페이스로 시각화

## 2. Hubble 설치 및 설정

Cilium 설치 후 helm upgrade 또는 cilium CLI 명령을 사용하여 Hubble을 활성화할 수 있습니다.

### Hubble 활성화

#### 방법 1: helm upgrade 사용

Hubble UI 활성화, NodePort 타입으로 서비스 노출, Prometheus 메트릭 활성화를 동시에 진행할 수 있습니다.

```bash
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
  --set hubble.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.service.type=NodePort \
  --set hubble.ui.service.nodePort=31234 \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload}"
```

#### 방법 2: cilium CLI 사용

간단하게 Hubble과 UI를 활성화할 수 있습니다.

```bash
cilium hubble enable --ui
```

### Hubble UI 접속

UI 서비스가 NodePort로 설정되면 아래 명령으로 접속 주소를 확인할 수 있습니다.

```bash
NODEIP=$(ip -4 addr show eth1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo -e "http://$NODEIP:31234"
```

웹 브라우저로 해당 주소에 접속한 후, kube-system 네임스페이스를 선택하여 서비스 맵을 확인합니다.

### Hubble CLI 사용 및 API 접근

#### 로컬 PC에 Hubble CLI 설치

로컬 환경에 맞는 Hubble CLI 바이너리를 다운로드하여 설치합니다.

#### 포트 포워딩을 통한 API 접속

로컬 머신에서 클러스터의 Hubble Relay 서비스로 포트 포워딩을 설정하여 Hubble API에 접근할 수 있습니다.

```bash
cilium hubble port-forward &
```

포트 포워딩 후 `hubble status` 명령으로 API 접근을 확인할 수 있습니다.

```bash
hubble status
# Healthcheck (via localhost:4245): Ok
# Current/Max Flows: 12,285/12,285 (100.00%)
# Flows/s: 41.28
```

## 3. Hubble 활용 실습 (Star Wars 데모)

### 데모 애플리케이션 배포

Star Wars 예제 애플리케이션(deathstar, tiefighter, xwing)을 배포하여 트래픽을 생성합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/http-sw-app.yaml
```

### L3/L4 정책 적용 및 관찰

#### 정책 미적용 상태 확인

초기에는 네트워크 정책이 없으므로 xwing과 tiefighter 모두 deathstar 서비스에 접근할 수 있습니다. `hubble observe` 명령으로 트래픽을 모니터링합니다.

```bash
# xwing에서 deathstar로 호출
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

# tiefighter에서 deathstar로 호출
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

Hubble UI에서도 xwing과 tiefighter가 deathstar로 연결되는 것을 확인할 수 있습니다.

#### L3/L4 정책 적용

`org=empire` 레이블을 가진 파드만 deathstar에 접근하도록 L3/L4 CiliumNetworkPolicy를 적용합니다. 이 정책은 tiefighter의 접근은 허용하지만, xwing의 접근은 차단합니다.

```yaml
# sw_l3_l4_policy.yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L3-L4 policy to restrict deathstar access to empire ships only"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: "TCP"
```

```bash
# 정책 적용
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/sw_l3_l4_policy.yaml
```

#### 정책 적용 후 관찰

다시 xwing에서 deathstar로의 접근을 시도하면 타임아웃이 발생합니다.

```bash
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing --connect-timeout 2
```

Hubble CLI로 차단된(DROPPED) 패킷을 확인할 수 있습니다.

```bash
hubble observe -f --type drop
```

### L7 정책 적용 및 관찰

#### L7 정책의 필요성

L3/L4 정책만으로는 같은 `org=empire` 레이블을 가진 tiefighter가 허용되지 않은 API(예: PUT /v1/exhaust-port)를 호출하는 것을 막을 수 없습니다.

#### L7 (HTTP) 정책 적용

기존 정책(rule1)을 수정하여 tiefighter가 POST /v1/request-landing API만 호출하도록 제한하는 L7 정책을 추가합니다.

```yaml
# sw_l3_l4_l7_policy.yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L7 policy to restrict access to specific HTTP call"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/v1/request-landing"
```

```bash
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/sw_l3_l4_l7_policy.yaml
```

#### 정책 적용 후 관찰

tiefighter에서 허용되지 않은 exhaust-port로의 PUT 요청을 보내면 "Access denied" 응답을 받습니다.

```bash
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
```

Hubble에서 해당 요청이 정책에 의해 차단(DROPPED)되었음을 L7 수준에서 확인할 수 있습니다.

```bash
hubble observe -f --pod deathstar --verdict DROPPED
# ... http-request DROPPED (HTTP/1.1 PUT http://deathstar.default.svc.cluster.local/v1/exhaust-port)
```

## 4. Hubble Exporter (흐름 로그)

Hubble Exporter는 Hubble의 흐름(flow) 로그를 파일에 저장하는 기능입니다. 파일 로테이션, 크기 제한, 필터링을 지원합니다.

### 설정 및 확인

Helm 업그레이드 시 관련 플래그를 설정하여 활성화할 수 있습니다. 이미 실습 환경에서는 설정되어 있습니다.

```bash
# helm upgrade ...
  --set hubble.export.static.enabled=true \
  --set hubble.export.static.filePath=/var/run/cilium/hubble/events.log
```

Cilium 파드 내에서 로그 파일이 정상적으로 생성되는지 확인할 수 있습니다.

```bash
kubectl -n kube-system exec ds/cilium -- tail -f /var/run/cilium/hubble/events.log | jq
```

### 성능 튜닝 및 동적 설정

#### 성능 튜닝
- `allowList`, `denyList`, `fieldMask` 같은 옵션을 사용하여 저장되는 로그의 양을 조절
- 성능에 미치는 영향을 최소화할 수 있음

#### 동적 설정
- Cilium 파드를 재시작하지 않고도 흐름 로그 설정을 변경할 수 있는 동적 설정을 지원
- ConfigMap을 통해 관리됨

## 결론

Hubble은 Cilium과 eBPF를 기반으로 하여 Kubernetes 클러스터의 네트워크를 효과적으로 관찰하고 정책을 적용할 수 있는 강력한 도구입니다. L3/L4부터 L7까지의 네트워크 정책을 세밀하게 제어할 수 있으며, 실시간 트래픽 모니터링과 보안 관측을 통해 클러스터의 네트워크 상태를 완전히 파악할 수 있습니다. 