# 10. 생태계 / GitOps

CKA 시험 범위는 아니지만 **실무에서 흔히 쓰는 생태계 도구** 정리. 특히 GitOps 기반 배포(ArgoCD).

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
