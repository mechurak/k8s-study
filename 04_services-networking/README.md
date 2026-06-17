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
- **oauth2-proxy** — 인증 게이트 + path 라우팅, ingress auth annotation, `--upstream` 경로 동작 (실무) → [oauth2-proxy.md](./oauth2-proxy.md)
- **cert-manager** — TLS 인증서 자동 발급/갱신, ACME DNS-01(Cloudflare·사내 egress proxy 경유) (실무) → [cert-manager.md](./cert-manager.md) *(준비 중)*
- **external-dns** — Ingress/Service → DNS 레코드 자동 관리, opt-in annotation·upsert-only (실무) → [external-dns.md](./external-dns.md) *(준비 중)*
- **소스 IP 화이트리스트** — `loadBalancerSourceRanges`(Service) / `whitelist-source-range`(Ingress), admin UI 접근통제 → [external-access.md](./external-access.md#화이트리스트는-어디에-거나--두-위치)
- **NetworkPolicy** — ingress/egress 규칙, default deny
  - kind 기본 CNI(kindnet)는 정책 미적용 → Calico 설치 필요 (`01_lab-environment/kind.md` 참고)

## 정리

> (학습하며 채워나갈 자리)

## 참고

- [Services, Load Balancing, Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
