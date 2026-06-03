# 10. 생태계 / GitOps

CKA 시험 범위는 아니지만 **실무에서 흔히 쓰는 생태계 도구** 정리. 특히 GitOps 기반 배포(ArgoCD).

> Helm·Kustomize는 CKA 범위라 `06_cluster-ops`에 있다. 여기서는 그것들을 활용하는 **상위 워크플로(GitOps/CD)** 에 집중.

## 다루는 내용
### ArgoCD (GitOps CD)
- GitOps 개념 — Git을 단일 소스로, 선언적 배포, 자동 동기화
- ArgoCD 아키텍처 — Application, Project, repo-server, application-controller
- Application 정의 — source(repo/path/helm/kustomize) + destination(cluster/namespace)
- Sync 정책 — manual vs automated, self-heal, prune
- App of Apps 패턴
- Helm/Kustomize와 연동 (ArgoCD가 렌더링)
- (실무) EKS + ArgoCD 구성, SSO/RBAC

### (추후) 더 볼 것
- Flux (ArgoCD 대안 GitOps)
- Kustomize/Helm 실무 패턴 심화
- 관측성 스택 (Prometheus, Grafana) — 필요 시

## 정리

> (학습/실무하며 채워나갈 자리)

## 참고

- [Argo CD 공식 문서](https://argo-cd.readthedocs.io/)
- [GitOps (OpenGitOps)](https://opengitops.dev/)
- [Argo CD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
