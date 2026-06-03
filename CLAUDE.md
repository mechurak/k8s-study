# CLAUDE.md

이 저장소에서 작업할 때 Claude가 따를 가이드. (사용자와의 대화·정리는 **한국어**로)

## 이 저장소는 무엇인가

Kubernetes 학습 + **CKA 자격증** 취득 + **AWS EKS** 실무 운영을 목표로, 스터디하며 주요 내용을 차근차근 정리하는 개인 학습 저장소. 코드 프로젝트가 아니라 **마크다운 학습 노트** 모음이다.

- 사용자: macOS(Apple Silicon, M3 Pro) 환경. 업무는 AWS EKS 운영.
- 로컬 실습 환경: **colima(런타임) + kind(멀티노드 클러스터)**. 자세한 건 `01_lab-environment/kind.md`.

## 폴더 구조 (번호 = 학습 순서 = CKA 도메인)

| 폴더 | 주제 | CKA 도메인 / 성격 |
|------|------|------|
| `01_lab-environment` | colima·kind·killercoda·EKS 환경 준비 | 환경 |
| `02_core-concepts` | Pod, Deployment, kubectl 기본 | 기초 |
| `03_workloads-scheduling` | 워크로드, 스케줄링, ConfigMap/Secret, 오토스케일링 | Workloads & Scheduling (~15%) |
| `04_services-networking` | Service, Ingress, Gateway API, NetworkPolicy, DNS | Services & Networking (~20%) |
| `05_storage` | Volume, PV/PVC, StorageClass | Storage (~10%) |
| `06_cluster-ops` | kubeadm, 업그레이드, etcd, RBAC, HA, **Helm/Kustomize**, CRD | Cluster Architecture (~25%) |
| `07_troubleshooting` | 노드/파드/네트워크/컨트롤플레인 진단 | Troubleshooting (~30%) |
| `08_cka-prep` | 치트시트, killer.sh, 기출 패턴, 시험 팁 | 시험 대비 |
| `09_aws-eks` | EKS 생성·운영, 실무 메모 | 실무 |
| `10_ecosystem-gitops` | ArgoCD(GitOps) 등 생태계 도구 | 실무 (CKA 외) |

- 루트 `README.md`가 전체 인덱스(목표·구조·로드맵)다. 폴더를 추가/변경하면 **루트 README의 폴더표·로드맵도 함께 갱신**한다.

## 문서 컨벤션

각 폴더의 `README.md`는 다음 형식을 따른다 — 새 내용을 추가할 때도 이 틀을 유지한다.

```markdown
# NN. 제목
> CKA 도메인: **도메인명 (~비중%)**   ← 해당될 때만

## 학습 목표
- [ ] 항목 (체크박스로, 학습하면 [x]로 체크)

## 정리
> (학습하며 채워나갈 자리)   ← 여기에 실제 학습 내용을 정리

## 실습 매니페스트   ← YAML이 있을 때만
- `manifests/deployment.yaml` — 한 줄 설명

## 참고
- [공식 문서 링크](...)
```

- **학습 목표**: 체크리스트. 학습을 마치면 `- [ ]` → `- [x]`로 바꾼다.
- **정리**: 실제 스터디 내용이 쌓이는 곳. 개념 설명 + 실습 명령/매니페스트 + 직접 겪은 점 위주로.
- 폴더 간 관련 내용은 상대 경로로 교차 링크한다 (예: `04`의 NetworkPolicy → `01_lab-environment/kind.md`의 Calico 안내).
- 트러블슈팅성 내용은 **증상 → 원인 → 해결** 표로 쌓으면 좋다 (`kind.md` 트러블슈팅 표 참고).

### 실습 매니페스트(YAML) 규칙

- 각 주제 폴더 안에 **`manifests/` 하위폴더**를 두고 YAML을 둔다(설명 README와 동거). 루트에 모으지 않는다.
- 파일명은 **리소스/목적이 드러나게** 짓는다: `deployment.yaml`, `service-clusterip.yaml`, `networkpolicy-deny-all.yaml`.
- 파일 상단에 **무엇을/왜인지 한 줄 주석**을 단다. `kubectl apply -f <file>`로 바로 돌아가게 유지.
- 해당 폴더 README의 `## 실습 매니페스트`에 파일 목록과 한 줄 설명을 적어 연결한다.
- 이미지 태그는 `latest` 대신 고정 버전(예: `nginx:1.27`)을 쓴다.

## 핵심 결정 사항 (헷갈리지 말 것)

- **Helm·Kustomize → `06`** (공식 커리큘럼상 Cluster Architecture "install cluster components"). 03 아님.
- **ArgoCD/GitOps → `10`** (CKA 범위 아님, 실무 도구).
- **로컬 메인은 kind**, colima는 보조(런타임 제공). minikube/k3s는 보조·비교용.
- **kubeadm 업그레이드·etcd 백업/복구 반복 연습은 killercoda**(무료 멀티노드)로 충분. `kubeadm init` 0→1 구축만 Multipass VM으로 1회.
- **CKA 학습은 EKS에서 하지 않는다** — 관리형이라 control plane(etcd/kubeadm)을 못 만진다. EKS는 실무 관점(`09`)으로 분리.

## 실습 환경 명령 (요약)

```bash
# 최초 1회 (사양 플래그는 이때만, 내장 k3s는 끔)
colima start --cpu 4 --memory 8 --disk 60 --kubernetes=false
kind create cluster --name study

# 매번: colima start (시작) / colima stop (종료, 데이터 보존)
```

자세한 건 `01_lab-environment/kind.md` (설치·구성·일상 운영·트러블슈팅).

## 정보 정확성

- CKA는 분기마다 k8s 버전·세부 항목이 바뀐다. 커리큘럼 관련 내용을 추가/검증할 때는 **공식 출처**를 확인한다:
  - [CKA 커리큘럼 (cncf/curriculum)](https://github.com/cncf/curriculum)
  - [CKA 변경 내역 (Linux Foundation)](https://training.linuxfoundation.org/certified-kubernetes-administrator-cka-program-changes/)
- 현재 문서는 **2025-02-18 개정 커리큘럼** 기준으로 정렬돼 있다.
- 외부 사실(버전·비중·도구 동작)을 단정하기 전에 확인하고, 출처 링크를 남긴다.
