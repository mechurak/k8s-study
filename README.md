# k8s-study

Kubernetes 학습 + **CKA 자격증** 취득 + **AWS EKS** 실무 운영을 목표로 차근차근 정리하는 저장소.

## 🎯 목표

1. **k8s 기본 학습** — 핵심 오브젝트와 동작 원리 이해
2. **CKA 자격증 취득** — 시험 범위(특히 클러스터 라이프사이클) 대비
3. **실무 EKS 운영** — AWS EKS 환경에서의 운영 감각

## 📁 폴더 구조

CKA 시험 도메인에 맞춰 정렬했다. 괄호 안 %는 **CKA 출제 비중**(2025-02-18 개정 기준 — [공식 커리큘럼](https://github.com/cncf/curriculum) · [변경 내역](https://training.linuxfoundation.org/certified-kubernetes-administrator-cka-program-changes/)).

| 폴더 | 내용 | CKA 도메인 |
|------|------|-----------|
| [01_lab-environment](./01_lab-environment/) | colima·kind·killercoda·EKS 환경 준비 | — |
| [02_core-concepts](./02_core-concepts/) | Pod, ReplicaSet, Deployment, kubectl 기본 | 기초 |
| [03_workloads-scheduling](./03_workloads-scheduling/) | Deployment, Job/CronJob, 스케줄링, ConfigMap/Secret | Workloads & Scheduling (~15%) |
| [04_services-networking](./04_services-networking/) | Service, Ingress, Gateway API, NetworkPolicy, DNS | Services & Networking (~20%) |
| [05_storage](./05_storage/) | Volume, PV/PVC, StorageClass | Storage (~10%) |
| [06_cluster-ops](./06_cluster-ops/) | 설치, 업그레이드, etcd 백업·복구, RBAC, Helm/Kustomize | Cluster Architecture, Install & Config (~25%) |
| [07_troubleshooting](./07_troubleshooting/) | 노드/파드/네트워크/컨트롤플레인 문제 진단 | Troubleshooting (~30%) |
| [08_cka-prep](./08_cka-prep/) | 치트시트, killer.sh, 기출 패턴, 시험 팁 | 시험 대비 |
| [09_aws-eks](./09_aws-eks/) | EKS 클러스터 생성·운영, 실무 메모 | 실무 |
| [10_ecosystem-gitops](./10_ecosystem-gitops/) | ArgoCD(GitOps), 생태계 도구 | 실무 (CKA 외) |

> Helm·Kustomize는 06번(공식 커리큘럼상 Cluster Architecture)에, ArgoCD 등 GitOps 도구는 10번에 정리한다.

## 🗺️ 학습 로드맵

대략 7주 기준. 주차는 폴더와 매핑된다. (환경 준비는 [01번](./01_lab-environment/) 참고)

1. **1~2주 — 기초** (02번): kind + killercoda로 핵심 오브젝트(Pod, Deployment, Service, ConfigMap) 손에 익히기
2. **3~5주 — 심화** (03~05번): 워크로드·스케줄링, 서비스·네트워킹, 스토리지. Gateway API / NetworkPolicy / 오토스케일링 포함
3. **6~7주 — 클러스터 운영 + 트러블슈팅** (06~07번): CKA 득점 포인트
   - RBAC, Helm/Kustomize, CRD·오퍼레이터 등 클러스터 관리 영역
   - killercoda로 업그레이드 / etcd 백업·복구 반복 숙달 (10회+, VM 불필요)
   - Multipass VM에서 `kubeadm init`부터 클러스터 1회 직접 구축 (라이프사이클 이해용)
4. **시험 직전 — CKA 마무리** (08번): killer.sh 2세션 풀고 틀린 것 복습
5. **병행 / 이후 — 실무** (09~10번): EKS 직접 배포, ArgoCD/GitOps

## 🛠️ 실습 환경 (요약)

- **로컬 메인**: colima(런타임) + kind(멀티노드 클러스터) — [상세](./01_lab-environment/kind.md)
- **문제풀이**: [killercoda](https://killercoda.com/cka)(무료) + killer.sh(시험 등록 시 포함)
- **실무**: AWS EKS

```bash
# 최초 1회
colima start --cpu 4 --memory 8 --disk 60 --kubernetes=false
kind create cluster --name study
# 매번: colima start (시작) / colima stop (종료)
```

## 📚 참고 자료

- [Kubernetes 공식 문서](https://kubernetes.io/docs/)
- [CKA – Linux Foundation](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)
- [killer.sh CKA Simulator](https://killer.sh/cka) · [killercoda CKA](https://killercoda.com/cka)
- [kind 공식 문서](https://kind.sigs.k8s.io/)
