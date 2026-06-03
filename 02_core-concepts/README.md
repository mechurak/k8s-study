# 02. 핵심 개념 (기초)

k8s를 다루기 위한 가장 기본이 되는 오브젝트와 `kubectl` 사용법. 이후 모든 주제의 토대.

> 📘 **실습은 [practice.md](./practice.md)** 가이드를 따라 직접 진행한다.

## 다루는 내용

주제별 개념 문서로 정리돼 있다.

- **[클러스터 구조](./cluster-architecture.md)** — control plane vs node, 핵심 컴포넌트
- **[kubectl 기본 & 매니페스트 구조](./kubectl-basics.md)** — 명령 구조, imperative vs declarative, `--dry-run`, `explain`, `apiVersion/kind/metadata/spec`
- **[Pod](./pods.md)** — 라이프사이클, probe, 멀티 컨테이너
- **[ReplicaSet & Deployment](./deployments.md)** — 자기복구, 롤아웃/롤백, 업데이트 전략
- **[Label·Selector & Namespace](./labels-namespaces.md)** — 리소스 분류와 격리

## 실습 매니페스트

- [`manifests/pod.yaml`](./manifests/pod.yaml) — 가장 기본적인 Pod(nginx) 하나
- [`manifests/deployment.yaml`](./manifests/deployment.yaml) — Pod 3개를 관리하는 Deployment (스케일/롤아웃 실습)

```bash
kubectl apply -f manifests/
kubectl get pod,deploy -l app=nginx
```

## 참고

- [Kubernetes 기본 개념](https://kubernetes.io/docs/concepts/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
