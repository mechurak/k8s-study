# 02. 핵심 개념 (기초)

k8s를 다루기 위한 가장 기본이 되는 오브젝트와 `kubectl` 사용법. 이후 모든 주제의 토대.

## 학습 목표

- [ ] 클러스터 구조 개요 (control plane vs node, 핵심 컴포넌트)
- [ ] `kubectl` 기본 (get/describe/apply/delete, `-o yaml`, `--dry-run=client`)
- [ ] Pod — 라이프사이클, 컨테이너/사이드카, probe(liveness/readiness/startup)
- [ ] Label & Selector, Annotation
- [ ] Namespace
- [ ] ReplicaSet
- [ ] Deployment — 롤아웃/롤백, 전략(RollingUpdate/Recreate)
- [ ] 매니페스트(YAML) 구조와 `apiVersion/kind/metadata/spec`

## 정리

> (학습하며 채워나갈 자리)

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
