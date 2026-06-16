# Gateway API — Ingress의 후속 표준 (역할 분리 · 고급 라우팅)

> CKA 도메인: **Services & Networking (~20%)**. 2025 개정에서 신규 추가. 선행 개념은 [ingress.md](./ingress.md).

## 한 줄 요약

> **Gateway API = Ingress가 부족했던 점(역할 분리·고급 라우팅·HTTP 외 프로토콜)을 처음부터 다시 설계한 공식 후속 표준.**

Ingress는 폐기된 게 아니다. 단순 HTTP 라우팅엔 여전히 충분하다. 다만 **새 투자·기능은 Gateway API로** 모이고 있고, 멀티팀·고급 트래픽 제어·non-HTTP가 필요하면 Gateway API가 답이다.

## 왜 Ingress를 두고 새로 만들었나

[ingress.md](./ingress.md)에 적은 Ingress의 한계들이 그대로 Gateway API의 설계 동기다:

| Ingress의 한계 | Gateway API의 해법 |
|---|---|
| 사실상 **HTTP(S) 전용** (TCP/UDP/gRPC 어려움) | `HTTPRoute`·`TCPRoute`·`GRPCRoute`·`TLSRoute` 등 **프로토콜별 리소스** |
| 고급 라우팅(헤더·메서드·가중치 분배)이 **표준에 없어 annotation 떡칠** | 헤더/메서드/쿼리 매칭, **트래픽 분배(가중치)** 가 **스펙 자체에** 들어감 |
| 리소스 1장을 **모두가 같이 수정** (역할 구분 없음) | **역할별로 리소스가 쪼개짐**(아래) — 큰 조직의 핵심 동기 |
| Controller마다 annotation이 달라 **이식성 ↓** | 동작이 **표준 스펙으로 정의** → 구현체 갈아타기 쉬움 |

## 가장 큰 차이 — 역할(role)로 리소스를 나눈다

Ingress는 한 리소스에 "인프라 노출 + 라우팅 규칙"이 섞여 있다. Gateway API는 이걸 **누가 소유하는가**로 3층으로 쪼갰다.

```mermaid
flowchart TD
    gc[GatewayClass<br/>'어떤 구현체로?' nginx/istio…] -->|인프라 제공자/벤더| g
    g[Gateway<br/>리스너: 80/443, TLS, 어떤 IP/포트로 노출] -->|플랫폼/인프라 팀 소유| r1
    g --> r2
    r1[HTTPRoute<br/>app.company.com/ → user-web] -->|앱 개발팀 소유| svc1[Service]
    r2[HTTPRoute<br/>/admin → admin-web] -->|다른 앱팀 소유| svc2[Service]
```

| 리소스 | 누가 소유 | 무엇을 정의 | Ingress로 치면 |
|---|---|---|---|
| **GatewayClass** | 인프라 제공자/벤더 | 어떤 구현체(nginx, istio, …)로 처리할지 | `IngressClass`와 유사 |
| **Gateway** | 플랫폼/인프라 팀 | 리스너(포트·프로토콜·TLS), 외부 노출 지점 | (Ingress엔 분리된 개념 없음) |
| **HTTPRoute** (등) | **앱 개발팀** | 호스트/경로 → 어느 Service로 | Ingress의 `rules` 부분 |

이 분리가 핵심이다. **인프라 팀이 Gateway(노출·TLS)를 한 번 만들어두면, 각 앱팀은 자기 `HTTPRoute`만 붙인다.** Ingress에서 "팀별로 쪼개기"를 annotation·host merge로 억지로 하던 걸, Gateway API는 **모델 차원에서** 지원한다. `HTTPRoute`는 다른 네임스페이스의 Gateway에도 attach 가능하며(cross-namespace), Gateway 쪽에서 `allowedRoutes`로 **어떤 네임스페이스/라벨의 Route를 받을지 권한 통제**한다.

## 라우팅 예시 — 헤더 매칭 + 가중치 분배

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: user-web
spec:
  parentRefs:
  - name: company-gateway          # 인프라팀이 만든 Gateway에 붙음
  hostnames: ["app.company.com"]
  rules:
  - matches:
    - path: { type: PathPrefix, value: / }
      headers:                       # ← Ingress 표준엔 없던 헤더 매칭
      - name: x-canary
        value: "true"
    backendRefs:
    - { name: user-web-canary, port: 80, weight: 10 }   # ← 가중치 트래픽 분배(카나리)
    - { name: user-web,        port: 80, weight: 90 }
```

카나리(canary)·블루그린·헤더 기반 분기를 **annotation 없이 표준 필드로** 한다는 게 큰 장점이다.

대응되는 Gateway(인프라팀이 한 번 만들어 둠):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: company-gateway
spec:
  gatewayClassName: nginx           # 어떤 구현체가 처리할지 (GatewayClass 이름)
  listeners:
  - name: web
    protocol: HTTP
    port: 80
    allowedRoutes:                  # 어떤 네임스페이스의 Route를 받을지 통제
      namespaces: { from: All }
```

## Ingress ↔ Gateway API 빠른 매핑

| Ingress | Gateway API |
|---|---|
| `IngressClass` | `GatewayClass` |
| (없음 — Ingress에 섞임) | `Gateway` (리스너·TLS·노출) |
| `Ingress`의 `rules` | `HTTPRoute` (그리고 TCP/GRPC/TLS Route) |
| Ingress Controller | Gateway 구현체(NGINX Gateway Fabric, Istio, Envoy Gateway …) |

## Ingress와 Gateway를 같이 쓸 수 있나 — 공존 / 마이그레이션

**된다. 둘은 완전히 독립된 API라 한 클러스터에서 동시 운영이 가능**하고, 마이그레이션 중인 클러스터는 거의 다 이 상태다.

| 형태 | 구성 | 진입점(LB/IP) |
|---|---|---|
| **별도 구현체 2개** | ingress-nginx(Ingress) + Envoy Gateway(Gateway) 등 따로 | **2개** (각자 LoadBalancer/NodePort) |
| **한 구현체가 둘 다 지원** | Istio·Cilium·NGINX은 Ingress·Gateway API **모두 구현** | 공유 가능(구현체 설정에 따라) |
| **마이그레이션 과도기** | 기존 Ingress 두고 새 라우트만 HTTPRoute로 | 보통 한동안 **2개** 운영 |

```mermaid
flowchart TB
    dns1["old.app.com"] --> ic["ingress-nginx<br/>(Ingress 처리)"] --> s1["Service A"]
    dns2["new.app.com"] --> gw["Gateway 구현체<br/>(HTTPRoute 처리)"] --> s2["Service B"]
```

### ⚠️ 공존 시 챙길 것

- **진입점이 둘이면 DNS·LB가 따로다.** `old.app.com`은 Ingress LB로, `new.app.com`은 Gateway LB로 — DNS가 어느 IP를 가리키는지 정확해야 한다. on-prem이면 LoadBalancer IP(또는 NodePort)가 **2개** 뜬다.
- **같은 host/port를 둘이 동시에 주장하면 충돌.** Gateway API는 *자기들끼리는* 우선순위 규칙이 있지만 **Ingress ↔ Gateway 사이엔 중재가 없다** — 실제 LB 리스너를 먼저 잡는 쪽이 이긴다. 같은 도메인을 양쪽에 두지 말 것.
- **한 구현체가 둘 다 처리**하면(Istio/Cilium 등) 데이터플레인·LB를 공유해 깔끔하다. **별도 구현체 2개**면 리소스·TLS 인증서·관측을 **두 벌** 관리하게 된다.

### 권장 방향 — 점진 이전

공식 권장은 빅뱅 교체가 아니라 **점진 이전**이다: 기존 Ingress 유지 → 새/이전 라우트를 HTTPRoute로 → 다 옮기면 Ingress 제거. [`ingress2gateway`](https://github.com/kubernetes-sigs/ingress2gateway) 툴이 기존 Ingress를 HTTPRoute로 변환해 준다([마이그레이션 가이드](https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/)).

> 🔎 **둘 다 쓰는 클러스터를 추적**할 땐 양쪽을 다 조회해야 전체가 보인다 — `kubectl get ingress -A` **와** `kubectl get gateway,httproute -A`. 그 뒤 `backend`/`backendRefs` → Service → EndpointSlice → Pod는 동일([연결 추적](#연결-추적--gateway가-내부-service에-어떻게-붙어-있나)).

## on-prem / 실무 관점

- **구현체(Controller)가 필요한 건 Ingress와 똑같다.** Ingress Controller 자리에 "Gateway 구현체"가 들어간다고 보면 된다: NGINX Gateway Fabric, Istio, Envoy Gateway, Cilium 등.
- **외부 노출 방식도 동일** — on-prem이면 [MetalLB](./metallb.md) / NodePort / 사내 LB로 Gateway를 노출한다([ingress.md](./ingress.md)의 노출 표와 같음).
- **CRD 설치가 선행**되어야 한다. Gateway API는 코어가 아니라 별도 CRD 묶음으로 배포된다(`kubectl apply` for standard channel CRDs) + 구현체.
- 확인 명령은 Ingress와 평행하다:
  ```bash
  kubectl get gatewayclass            # 설치된 Gateway 구현체
  kubectl get gateway -A              # 인프라팀이 만든 노출 지점(리스너/주소)
  kubectl get httproute -A            # 앱들의 라우팅
  kubectl describe httproute <name>   # 특정 Route의 매칭/백엔드/상태
  ```

## 연결 추적 — Gateway가 내부 service에 어떻게 붙어 있나

[oauth2-proxy.md "연결 추적"](./oauth2-proxy.md#연결-추적--ingressoauth2-proxy가-내부-service에-어떻게-붙어-있나)의 **Gateway API 버전**. 객체 모델이 달라 연결 고리가 바뀐다 — `Ingress.spec.rules`가 아니라 **HTTPRoute의 `backendRefs`**, 라우트↔게이트웨이는 **`parentRefs`** 로 묶인다.

```
Ingress 방식:  Ingress(spec.rules → backend)              → Service → EndpointSlice → Pod
Gateway 방식:  GatewayClass → Gateway → HTTPRoute(backendRefs) → Service → EndpointSlice → Pod
                              (parentRefs로 묶임)
```

```bash
# ① Gateway의 진입점(리스너/주소/host)
kubectl -n <ns> get gateway <gw> -o jsonpath=\
'{range .spec.listeners[*]}{.name} {.protocol}/{.port} host={.hostname}{"\n"}{end}'; echo

# ② 어떤 HTTPRoute가 이 Gateway에 붙었나(parentRefs) + 백엔드(backendRefs)
kubectl get httproute -A -o jsonpath=\
'{range .items[*]}{.metadata.namespace}/{.metadata.name}  parent={.spec.parentRefs[*].name}  ->  {range .spec.rules[*].backendRefs[*]}{.name}:{.port} {end}{"\n"}{end}'

# ③ 붙음/안붙음은 status로 — Accepted·ResolvedRefs가 True여야 실제 라우팅됨
kubectl -n <ns> get httproute <rt> \
  -o jsonpath='{range .status.parents[*]}{range .conditions[*]}{.type}={.status} {end}{"\n"}{end}'
```

→ `backendRefs`의 Service부터는 **Ingress 때와 동일**: Service → EndpointSlice → Pod (위 링크의 Step 3). EndpointSlice가 비면 502/503의 급소인 것도 같다.

### 시각화 툴 — Gateway API에서도 쓰나

| 툴 | Gateway API에서 |
|---|---|
| **Hubble UI** (CNI가 Cilium) | ✅ **완전 동일.** 트래픽(eBPF) 레이어라 Ingress냐 Gateway냐를 안 따짐. Cilium이 Gateway 구현체면 가장 잘 맞음 |
| **Headlamp** | ✅ Gateway API 리소스 뷰 지원(없어도 CRD로 조회). 관계 클릭 추적은 버전·플러그인에 따라 차이 |
| **OpenLens/Lens** | △ 오브젝트는 CRD로 다 보임. 전용 그래프 뷰는 약해 "관계 따라가기"는 손이 더 감 |
| **kubectl-graph** | ⚠️ ownerReference 기반이라 `parentRefs`/`backendRefs`(참조) **edge를 자동으로 못 그릴 수** 있음 — 오브젝트는 나와도 Gateway→Service 선이 끊길 수 있다 |

> 💡 oauth2-proxy도 Gateway API에서는 보통 **(A) 방식**을 annotation 대신 `HTTPRoute` + ext-auth(또는 구현체별 정책 CRD)로 붙인다. 방식은 쓰는 구현체(Cilium·Istio·Envoy Gateway 등)마다 달라 해당 문서 확인이 필요하다.

> ⚠️ 어떤 툴도 **사내 L4/L7 LB·방화벽·NAT**(인프라 레이어)까지는 못 그린다 → k8s 경계 안쪽 + 진입/egress 지점까지가 한계.

## 시험·실무 팁

- CKA 2025 신규 항목. **개념(GatewayClass→Gateway→HTTPRoute 3층 구조)과 HTTPRoute 기본 작성**을 기대. Ingress와의 매핑으로 외우면 빠르다.
- **버전 채널**: `standard`(GA, HTTPRoute 등)와 `experimental`(TCPRoute 등 일부)로 나뉜다. 클러스터에 깔린 CRD가 어느 채널인지 확인.
- **API 그룹은 `gateway.networking.k8s.io`** (Ingress는 `networking.k8s.io`). 헷갈리기 쉬움.
- Route의 **status 조건(Accepted/ResolvedRefs)** 으로 "Gateway에 잘 붙었나/백엔드를 찾았나"를 진단한다 — Ingress보다 상태 피드백이 풍부.

## 참고

- [Gateway API (공식)](https://gateway-api.sigs.k8s.io/)
- [Ingress에서 Gateway API로 마이그레이션](https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/)
- [Gateway API Implementations(구현체 목록)](https://gateway-api.sigs.k8s.io/implementations/)
- 선행 개념 → [ingress.md](./ingress.md)
