# 10. 생태계 / GitOps

CKA 시험 범위는 아니지만 **실무에서 흔히 쓰는 생태계 도구** 정리. GitOps 배포(ArgoCD), 서비스 메시(Istio) 등.

> Helm·Kustomize는 CKA 범위라 `06_cluster-ops`에 있다. 여기서는 그것들을 활용하는 **상위 워크플로(GitOps/CD)** 에 집중.

## 다루는 내용
### ArgoCD (GitOps CD) → [argocd.md](./argocd.md)
- GitOps 개념 — Git을 단일 소스로, push형 vs pull형 배포
- ArgoCD 아키텍처 — Application, Project, repo-server, application-controller
- Application 정의 — source(repo/path/helm/kustomize) + destination(cluster/namespace)
- Sync 정책 — manual vs automated, self-heal, prune
- App of Apps 패턴
- (실무) EKS + ArgoCD, Secret 관리(평문 금지), SSO/RBAC

**실습** → [practice.md](./practice.md): ArgoCD 설치 → guestbook 체험 → 내 저장소로 nginx 배포 → PostgreSQL(StatefulSet) 배포 → GitOps 워크플로(push·self-heal·prune) 실험.
**운영 레시피** → [recipes.md](./recipes.md): 개발·운영하며 자주 하는 작업(예: 개발 중 DB 초기화) 모음.

### GitOps 시크릿 관리 → [secrets-management.md](./secrets-management.md)
- **🗺️ 전체 그림** — 저장(Sealed Secrets)·배포(Reflector)·백업/키보호(SOPS·age)·DR 절차가 어떻게 맞물리나
- **Sealed Secrets** — 런타임 시크릿을 git에 암호화 저장(kubeseal, 비대칭키) → [sealed-secrets.md](./sealed-secrets.md) · **실습** → [practice-sealed-secrets.md](./practice-sealed-secrets.md)
- **SOPS / age** — DR 백업 + Sealed Secrets 마스터키 자체 암호화("키의 키") → [sops-age.md](./sops-age.md)
- **Reflector** — Secret을 네임스페이스 간 복제(예: Strimzi KafkaUser 시크릿) → [reflector.md](./reflector.md)
- **런타임 시크릿 DR 절차** — 복구 순서 런북 → [secrets-dr.md](./secrets-dr.md)
> k8s 기본 `Secret`(base64) 개념 자체는 [03_workloads-scheduling](../03_workloads-scheduling/). 여기선 그걸 **git에 안전히 두고 운영**하는 생태계 도구.

### 서비스 메시 (Istio·Envoy) → [service-mesh.md](./service-mesh.md)
- 서비스 메시 개념 — east-west 통신에 트래픽 제어·mTLS·관측을 앱 수정 없이 (↔ north-south는 `04` ingress/gateway)
- 데이터/컨트롤 플레인 — Envoy 사이드카 + istiod, Envoy와의 관계
- 사이드카 vs 앰비언트, 대안(Linkerd/Cilium/Consul), 도입 판단·EKS 실무

### (추후) 더 볼 것
- Flux (ArgoCD 대안 GitOps)
- Kustomize/Helm 실무 패턴 심화 (렌더링은 ArgoCD가, 도구 자체는 [`06_cluster-ops`](../06_cluster-ops/))
- 관측성 스택 (Prometheus, Grafana) — 필요 시

## 실습 매니페스트
> ArgoCD가 **Git에서 읽어** 배포하므로, 이 파일들은 push돼 있어야 한다(자세한 흐름은 [practice.md](./practice.md)).

- `manifests/web/deployment.yaml`, `service.yaml` — nginx 데모 워크로드(Pod 배포 체험용)
- `manifests/postgres/secret.yaml`, `service.yaml`, `statefulset.yaml` — PostgreSQL(상태 저장, PVC 영속)
- `manifests/argocd-apps/web-app.yaml`, `postgres-app.yaml` — 위를 가리키는 ArgoCD `Application`(선언형)

## 참고

- [Argo CD 공식 문서](https://argo-cd.readthedocs.io/)
- [GitOps (OpenGitOps)](https://opengitops.dev/)
- [Argo CD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
