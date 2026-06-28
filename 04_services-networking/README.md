# 04. 서비스와 네트워킹

> CKA 도메인: **Services & Networking (~20%)**

파드 간/외부 통신, 서비스 노출, 네트워크 정책. 2025 개정에서 **Gateway API**가 추가됐다.

## 다루는 내용
- **🗺️ 사내 on-prem 외부 노출 스택 — 전체 그림** (Calico→MetalLB→ingress-nginx→cert-manager+external-dns+OIDC) → [external-access.md](./external-access.md)
- 클러스터 네트워킹 모델 (파드 네트워크, CNI 개요)
- Service — ClusterIP / NodePort / LoadBalancer / ExternalName
- **MetalLB** — on-prem에서 `type: LoadBalancer`를 성립시키는 구현체 (L2/BGP) (실무) → [metallb.md](./metallb.md)
- Endpoints / EndpointSlice
- DNS — 서비스/파드 DNS 이름 규칙
- Ingress — 규칙, 호스트/경로 기반 라우팅, IngressClass, 1 Ingress ↔ N Service, on-prem 노출 → [ingress.md](./ingress.md)
- **Gateway API** — GatewayClass / Gateway / HTTPRoute, Ingress 후속 표준 (신규) → [gateway-api.md](./gateway-api.md)
- **oauth2-proxy** — 인증 게이트 + path 라우팅, ingress auth annotation, `--upstream` 경로 동작 (실무) → [oauth2-proxy.md](./oauth2-proxy.md) · **실습(kind+Keycloak)** → [practice-oauth2-proxy.md](./practice-oauth2-proxy.md)
- **cert-manager** — TLS 인증서 자동 발급/갱신, ACME DNS-01(Cloudflare·사내 egress proxy 경유) (실무) → [cert-manager.md](./cert-manager.md) *(준비 중)*
- **external-dns** — Ingress/Service → DNS 레코드 자동 관리, opt-in annotation·upsert-only (실무) → [external-dns.md](./external-dns.md) *(준비 중)*
- **소스 IP 화이트리스트** — `loadBalancerSourceRanges`(Service) / `whitelist-source-range`(Ingress), admin UI 접근통제 → [external-access.md](./external-access.md#화이트리스트는-어디에-거나--두-위치)
- **NetworkPolicy** — ingress/egress 규칙, default deny
  - kind 기본 CNI(kindnet)는 정책 미적용 → Calico 설치 필요 (`01_lab-environment/kind.md` 참고)

## 실습 매니페스트

- **oauth2-proxy 랩** (kind + Keycloak, → [practice-oauth2-proxy.md](./practice-oauth2-proxy.md))
  - `manifests/oauth2-proxy/keycloak.yaml` — Keycloak(OIDC IdP), ingress 뒤 HTTP 노출
  - `manifests/oauth2-proxy/echo-apps.yaml` — 데모 frontend/backend(요청 echo)
  - `manifests/oauth2-proxy/oauth2-proxy-modeA.yaml` — (A) ingress auth_request 모드
  - `manifests/oauth2-proxy/ingress-modeA.yaml` — 보호 앱 + `/oauth2` ingress
  - `manifests/oauth2-proxy/oauth2-proxy-modeB.yaml` — (B) `--upstream` 직접 프록시 모드
  - `manifests/oauth2-proxy/ingress-modeB.yaml` — app host 전체 → oauth2-proxy

## 정리

> Services & Networking은 CKA 비중 **최대(~20%)**. 여기선 먼저 **Service의 종류와 Headless Service**를 정리한다. Ingress·Gateway API·oauth2-proxy 등은 위 개념 문서 링크 참고.

### Service 종류

파드는 죽고 다시 뜨며 IP가 바뀐다. **Service**는 파드 집합 앞에 **안정적인 접근점**을 두고, 라벨 셀렉터로 고른 파드(=Endpoints)에 트래픽을 보낸다.

| 타입 | 접근 범위 | 설명 |
|---|---|---|
| **ClusterIP** (기본) | 클러스터 내부 | 가상 IP(VIP) 하나로 내부 파드들에 로드밸런싱 |
| **NodePort** | 외부(노드IP:포트) | 모든 노드의 30000–32767 포트를 열어 외부 노출 |
| **LoadBalancer** | 외부(클라우드 LB) | 클라우드 LB 프로비저닝. on-prem은 [MetalLB](./metallb.md) 필요 |
| **ExternalName** | DNS 별칭 | 셀렉터 없이 외부 도메인으로 CNAME만 |
| **Headless** | (아래) | `clusterIP: None` — VIP·로드밸런싱 없이 파드 IP를 DNS로 직접 노출 |

```bash
kubectl expose deployment web --port=80 --target-port=8080        # ClusterIP
kubectl expose deployment web --type=NodePort --port=80
kubectl get svc,endpoints                                          # Service ↔ Endpoints 매핑 확인
```

### Headless Service — 파드를 콕 집어 부른다

`spec.clusterIP: None` 으로 만든 Service. **가상 IP와 로드밸런싱을 빼버린** 형태다.

- **일반 ClusterIP**: 클라이언트 → `[VIP 1개]` → (랜덤 분배) → pod-A/B/C. *어느 파드로 가는지 모른다.*
- **Headless**: DNS를 조회하면 **파드들의 실제 IP 목록**을 그대로 돌려준다. StatefulSet과 짝지으면 **파드마다 개별 DNS 이름**까지 생긴다.

```bash
# Headless + StatefulSet 일 때
nslookup clickhouse.data.svc.cluster.local                # → pod-0, pod-1 ... 모든 파드 IP
nslookup clickhouse-0.clickhouse.data.svc.cluster.local   # → pod-0 만 콕 집어
```

| | 일반 ClusterIP | Headless |
|---|---|---|
| `clusterIP` | 가상 IP 있음 | **`None`** |
| 동작 | 1개 VIP로 **로드밸런싱** | 파드 IP들을 **DNS로 직접 노출** |
| 파드 지목 | 불가 | **가능**(파드별 DNS) |
| 주 용도 | 무상태 앱(웹·API) | **StatefulSet**(DB·큐) |

**왜 DB(StatefulSet)에 쓰나**: 복제 클러스터에선 *"0번(프라이머리)에 써라"* 처럼 **특정 파드를 지목**해야 한다. VIP로 로드밸런싱되면 곤란하다. 그래서 파드마다 고정 DNS를 주는 Headless가 필수.

- StatefulSet 워크로드 개념 → [`03_workloads-scheduling`](../03_workloads-scheduling/)
- **실전(ClickHouse를 Headless+StatefulSet으로 배포, `nslookup`으로 파드별 DNS 확인)** → [`12_data-stores/clickhouse.md`](../12_data-stores/clickhouse.md) 3절 · [`practice-clickhouse.md`](../12_data-stores/practice-clickhouse.md) 2절

### DNS 이름 규칙 (요약)

- Service: `<svc>.<namespace>.svc.cluster.local`
- Headless 뒤 StatefulSet 파드: `<pod>.<svc>.<namespace>.svc.cluster.local`
- 같은 네임스페이스면 `<svc>` 만으로도 접근.

## 참고

- [Services, Load Balancing, Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
