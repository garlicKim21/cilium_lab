# [Cilium Study] 2주차 - 1. Cilium Hubble을 이용한 네트워크 관측

지난 [1주 차 포스팅](https://tech-recipe.tistory.com/37)에서는 Cilium과 eBPF의 기본 개념을 맛보고, 복잡한 iptables의 세계에서 벗어날 수 있다는 희망을 보았습니다. 하지만 CNI를 설치하고 클러스터가 잘 동작한다고 해서 모든 것을 파악하고 있다고 말할 수 있을까요? 사실 아직은 안갯속을 걷는 기분입니다. 파드 간에 어떤 통신이 오가는지, 어떤 정책이 실제로 적용되고 있는지 눈으로 직접 확인하고 싶다는 갈증이 생기기 마련이죠.

바로 이럴 때 필요한 것이 **관측 가능성(Observability)** 이고, Cilium의 세계에서는 그 해결사가 바로 **Hubble(허블)** 입니다. 이번 포스팅에서는 Cilium의 단짝 친구, Hubble을 설치하고 직접 사용해보면서 쿠버네티스 네트워크를 속속들이 들여다보는 시간을 가져보겠습니다.

---

## 1. 그래서, Hubble이 뭔가요?

Hubble은 Cilium과 eBPF를 기반으로 만들어진 **분산 네트워킹 및 보안 관측 플랫폼**입니다. 이름처럼 우주를 관측하는 허블 망원경같이, 복잡한 쿠버네티스 클러스터 내부의 네트워크 흐름을 훤히 들여다볼 수 있게 해줍니다. eBPF의 능력을 활용하기 때문에, 성능 저하(오버헤드)는 최소화하면서도 아주 깊은 수준의 가시성을 확보할 수 있다는 게 가장 큰 장점입니다.

### Hubble로 무엇을 할 수 있을까?

Hubble이 있으면 다음과 같은 질문에 대한 답을 명쾌하게 얻을 수 있습니다.

* **서비스 의존성 및 통신 맵:** "클러스터 안에서 어떤 서비스들이 서로 통신하고 있지?", "HTTP API 호출이나 Kafka 토픽 같은 L7 트래픽은 어떻게 흘러가고 있을까?" 와 같은 서비스 간의 의존성을 한눈에 파악할 수 있는 지도를 그려줍니다.
* **네트워크 모니터링 및 알림:** "최근 5분간 DNS 응답에 실패한 서비스는 뭐지?", "TCP 연결이 자꾸 끊기는 파드가 있던데, 원인이 뭘까?" 처럼 네트워크 통신의 실패와 그 원인을 추적할 수 있습니다.
* **애플리케이션 모니터링:** "특정 서비스에서 HTTP 500 에러 비율이 급증했는데?", "API 응답 시간의 99 백분위수는 어느 정도일까?" 와 같이 애플리케이션 레벨의 성능 지표를 확인할 수 있습니다.
* **보안 관측:** "네트워크 정책 때문에 차단된 통신은 어떤 것들이지?", "클러스터 외부에서 우리 서비스에 접근한 IP는 어디야?" 등 보안 관점에서의 흐름을 감시할 수 있습니다.

### Hubble을 이루는 친구들

Hubble은 몇 가지 구성 요소가 함께 작동하며 이런 마법 같은 일들을 처리합니다.

* **Hubble API:** 각 노드의 Cilium 에이전트 안에서 관찰된 트래픽 정보를 제공하는 핵심 API입니다.
* **Hubble CLI:** 터미널에서 Hubble API에 정보를 요청하고 결과를 받아보는 명령어 도구입니다. Cilium 에이전트 파드에 기본적으로 설치되어 있습니다.
* **Hubble Relay:** 각 노드에 분산된 Hubble API 데이터를 한곳으로 모아, 클러스터 전체의 네트워크 가시성을 제공하는 서비스입니다.
* **Hubble UI:** Relay가 모아준 데이터를 우리가 보기 편한 웹 인터페이스(특히 서비스 맵!)로 시각화해주는 고마운 친구입니다.

---

## 2. Hubble 설치하고 접속하기

Cilium이 이미 설치되어 있다면, Hubble을 활성화하는 것은 그리 어렵지 않습니다. `helm upgrade`나 `cilium` CLI 명령어로 간단하게 설정할 수 있습니다.

### Hubble 활성화

#### 방법 1: helm upgrade 사용하기
가장 확실하고 다양한 옵션을 한 번에 적용할 수 있는 방법입니다. UI, Relay, Prometheus 연동까지 한 번에 활성화해보겠습니다.

```bash
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.ui.service.type=NodePort \
  --set hubble.ui.service.nodePort=31234 \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload}"
```

#### 방법 2: cilium CLI 사용하기
간단하게 Hubble과 UI만 빠르게 띄워보고 싶을 때 유용합니다.

```bash
cilium hubble enable --ui
```

### Hubble UI 접속하기
이제 웹 브라우저로 Hubble UI에 접속해볼 시간입니다. NodePort로 서비스를 노출시켰으니, 아래 명령어로 접속 주소를 쉽게 확인할 수 있죠.

```bash
NODEIP=$(ip -4 addr show eth1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo -e "http://$NODEIP:31234"
```

출력된 주소로 접속해서 `kube-system` 네임스페이스를 선택하면, 드디어 영롱한 서비스 맵을 마주하게 될 겁니다!

### 내 PC에서 Hubble CLI 사용하기

파드에 접속하지 않고 로컬 PC에서 바로 `hubble` 명령어를 사용하면 훨씬 편리하겠죠? 그러기 위해선 약간의 준비가 필요합니다.

1.  **로컬 PC에 Hubble CLI 설치:** 운영체제에 맞는 바이너리를 다운로드하여 설치합니다.
2.  **포트 포워딩으로 API 연결:** 아래 명령어로 클러스터의 Hubble Relay 서비스와 내 PC를 연결해줍니다. `&`를 붙여 백그라운드로 실행하는 센스를 발휘해줍시다.

    ```bash
    cilium hubble port-forward &
    ```

포트 포워딩이 성공했다면, `hubble status` 명령어로 연결 상태를 확인할 수 있습니다.

```bash
hubble status
# Healthcheck (via localhost:4245): Ok
# Current/Max Flows: 12,285/12,285 (100.00%)
# Flows/s: 41.28
```
이제 모든 준비가 끝났습니다!

---

## 3. 백문이 불여일견, Hubble 실습 (feat. Star Wars)

개념 설명만으로는 감이 잘 오지 않죠. Cilium의 단골 예제인 스타워즈 데모 앱을 배포해서 Hubble의 진가를 직접 확인해보겠습니다.

### 데모 애플리케이션 배포
영화에 등장하는 `deathstar`, `tiefighter`, `xwing` 파드를 생성하여 테스트용 트래픽을 만들어 보겠습니다.

```bash
kubectl apply -f [https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/http-sw-app.yaml](https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/http-sw-app.yaml)
```

### L3/L4 정책: "우리 편만 들어와!"

#### 정책 적용 전
아무런 네트워크 정책이 없는 초기 상태에서는 제국군(`tiefighter`)이든 반란군(`xwing`)이든 모두 `deathstar`에 접근할 수 있습니다.

```bash
# 반란군(xwing)에서 데스스타 호출
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
# Ship landed

# 제국군(tiefighter)에서 데스스타 호출
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
# Ship landed
```
Hubble UI를 봐도 `xwing`과 `tiefighter`가 `deathstar`로 향하는 녹색 화살표가 잘 보일 겁니다.

#### L3/L4 정책 적용하기
이제 `org=empire` 레이블을 가진, 즉 제국군 소속 파드만 `deathstar`에 접근할 수 있도록 L3/L4 레벨의 네트워크 정책을 적용해 보겠습니다.

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
```bash
# 정책 적용
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/sw_l3_l4_policy.yaml
```

#### 정책 적용 후
다시 반란군 `xwing`이 접근을 시도하면, 이제는 응답 없이 타임아웃이 발생합니다.

```bash
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing --connect-timeout 2
# command terminated with exit code 28
```

이때 `hubble observe` 명령어로 차단(`DROPPED`)된 패킷을 확인하면, 정책에 의해 트래픽이 거부되었음을 명확히 알 수 있습니다. Hubble UI에서도 `xwing`에서 `deathstar`로 향하는 화살표가 붉은색으로 바뀌었을 겁니다.

```bash
hubble observe -f --type drop
```

### L7 정책: "허가된 임무만 수행해!"

#### L7 정책이 필요한 이유
L3/L4 정책으로 우리 편, 너희 편은 갈랐지만 아직 부족합니다. 같은 제국군 소속인 `tiefighter`가 허가되지 않은 행동, 예를 들어 `deathstar`의 배기구를 공격(`PUT /v1/exhaust-port`)하는 것을 막을 수는 없으니까요. 바로 이럴 때 L7 정책이 필요합니다.

#### L7 (HTTP) 정책 적용하기
기존 정책을 수정하여, `tiefighter`가 오직 착륙 요청(`POST /v1/request-landing`)만 할 수 있도록 L7 정책을 추가해 보겠습니다.

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
```bash
kubectl apply -f [https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/sw_l3_l4_l7_policy.yaml](https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/sw_l3_l4_l7_policy.yaml)
```

#### 정책 적용 후
이제 `tiefighter`로 허용되지 않은 배기구 공격을 시도하면, "Access denied" 라는 메시지를 받게 됩니다.

```bash
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
# Access denied
```

Hubble로 이 요청을 살펴보면, `DROPPED` 사유가 L7 정책 위반(`HTTP/1.1 PUT ...`)임을 정확하게 확인할 수 있습니다.

```bash
hubble observe -f --pod deathstar --verdict DROPPED
# ... http-request DROPPED (HTTP/1.1 PUT [http://deathstar.default.svc.cluster.local/v1/exhaust-port](http://deathstar.default.svc.cluster.local/v1/exhaust-port))
```
정말 강력하지 않나요? IP 주소나 포트 번호를 넘어, HTTP 메서드와 경로까지 제어할 수 있다니 놀랍습니다.

---

## 4. Hubble Exporter: 흐름 로그 저장하기

Hubble UI나 CLI는 실시간 모니터링에 아주 훌륭하지만, 나중에 분석하거나 감사(audit)를 위해 네트워크 흐름 기록을 파일로 남겨야 할 때도 있습니다. 이럴 때 사용하는 것이 바로 **Hubble Exporter**입니다.

### 설정 및 확인
Helm으로 설치할 때 관련 플래그를 설정하면 Exporter를 활성화할 수 있습니다. (사실 맨 처음 실행했던 helm 명령어에 이미 포함되어 있었습니다.)

```bash
# helm upgrade ...
  --set hubble.export.static.enabled=true \
  --set hubble.export.static.filePath=/var/run/cilium/hubble/events.log
```
Cilium 파드에 직접 들어가서 로그 파일이 잘 쌓이고 있는지 확인해볼 수 있습니다. `jq`를 이용하면 JSON 형식의 로그를 예쁘게 볼 수 있죠.

```bash
kubectl -n kube-system exec ds/cilium -- tail -f /var/run/cilium/hubble/events.log | jq
```

Exporter는 파일 크기나 시간에 따른 자동 로테이션, 특정 흐름만 기록하는 필터링(`allowList`, `denyList`) 등 유용한 기능들을 제공하니, 실제 운영 환경에서는 이런 튜닝 옵션들을 잘 활용하면 좋을 것 같습니다.

## 마무리하며

이번 주 스터디를 통해 Hubble을 직접 사용해보니, '관측 가능성'이라는 말이 피부로 와닿는 기분이었습니다. 막연하게만 느껴졌던 클러스터 내부의 네트워크 통신을 눈으로 직접 보고, 정책을 적용했을 때 트래픽이 어떻게 변하는지 실시간으로 추적할 수 있다는 점이 정말 매력적이었습니다. 마치 깜깜한 방에 불을 탁 켠 것처럼 시야가 환해지는 느낌이랄까요.

Cilium과 Hubble을 함께 사용한다면, 쿠버네티스 네트워킹은 더 이상 복잡하고 어려운 대상이 아니라, 우리가 완벽하게 제어하고 이해할 수 있는 영역이 될 수 있겠다는 확신이 들었습니다.

다음 포스팅에서는 또 다른 Cilium의 흥미로운 기능에 대해서 다뤄보도록 하겠습니다. 감사합니다.

