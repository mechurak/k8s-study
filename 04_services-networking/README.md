# 04. 서비스와 네트워킹

> CKA 도메인: **Services & Networking (~20%)**

파드 간/외부 통신, 서비스 노출, 네트워크 정책. 2025 개정에서 **Gateway API**가 추가됐다.

## 학습 목표

- [ ] 클러스터 네트워킹 모델 (파드 네트워크, CNI 개요)
- [ ] Service — ClusterIP / NodePort / LoadBalancer / ExternalName
- [ ] Endpoints / EndpointSlice
- [ ] DNS — 서비스/파드 DNS 이름 규칙
- [ ] Ingress — 규칙, 호스트/경로 기반 라우팅, IngressClass
- [ ] **Gateway API** — Gateway, HTTPRoute (신규)
- [ ] **NetworkPolicy** — ingress/egress 규칙, default deny
  - kind 기본 CNI(kindnet)는 정책 미적용 → Calico 설치 필요 (`01_lab-environment/kind.md` 참고)

## 정리

> (학습하며 채워나갈 자리)

## 참고

- [Services, Load Balancing, Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
