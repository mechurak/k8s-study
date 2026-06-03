# 01. 실습 환경 준비

k8s 스터디를 위한 실습 환경 정리 문서.

## 환경 구성 원칙

세 가지 목표(k8s 기초 · CKA · 실무 EKS — [루트 README](../README.md#-목표) 참고)는 각각 요구하는 환경이 다르다. 그래서 **하나의 환경에 다 욱여넣지 않고 용도별로 나눠서** 사용한다.

## 용도별 환경 매트릭스

| 목적 | 추천 환경 | 이유 |
|------|----------|------|
| CKA 문제풀이 감각 | killercoda + killer.sh | 시험과 동일한 터미널/UX, 무료~등록 시 포함 |
| 로컬 클러스터 실습 | **kind**(메인) + colima(보조) | 멀티노드를 가볍게, vanilla k8s에 가까움 |
| 클러스터 운영 실습(kubeadm) | Multipass/Lima VM 2~3대 | CKA 핵심인 직접 구축/업그레이드/etcd 백업·복구 |
| 실무 EKS | 실제 EKS (eksctl/Terraform) | 결국 업무 환경 그대로가 best |

---

## CKA 관점에서 먼저 짚을 것

CKA는 **performance-based**(실제 터미널에서 작업) 시험이다. 2025년 개정으로 출제 비중의 절반가량이 새 내용으로 바뀌었고, 환경 선택에 영향을 주는 부분은 다음과 같다.

- **kubeadm으로 직접 클러스터 구축 / 업그레이드 / etcd 백업·복구** → 단일 노드 k3s/colima로는 연습 불가. **멀티노드 + kubeadm**이 필요하다.
- 새로 강화된 영역: **Gateway API, Helm/Kustomize, NetworkPolicy, CRD, 트러블슈팅 비중 ↑**
- 시험은 보통 최신 k8s보다 1~2 버전 낮은 버전으로 진행된다.
- **등록하면 killer.sh 시뮬레이터 2회**(각 17문항, 36시간 액세스) 포함. 실제 시험보다 어렵기로 유명해 최고의 마무리 도구.

> ⚠️ colima/k3s만으로는 CKA의 "클러스터 라이프사이클" 영역을 채울 수 없다. 여기가 핵심.

---

## 각 옵션 평가

### colima `--kubernetes` (k3s)
- 👍 Mac에서 가장 가볍고 빠르게 뜸. Docker 런타임도 같이 해결.
- 👎 k3s는 경량 디스트로라 vanilla k8s와 차이가 있음(traefik 기본 탑재, sqlite/etcd, 단일 바이너리, 일부 컴포넌트 통합). **kubeadm 설치/업그레이드 연습 불가.**
- → **앱 배포 / 매니페스트 실습용 보조**로는 훌륭.

### kind (Kubernetes in Docker) — 로컬 메인 추천
- 👍 vanilla k8s, **멀티노드 클러스터를 config 한 장으로** 띄움. 빠르고 깨고 다시 만들기 쉬움. CNI/NetworkPolicy 실습 가능.
- 👎 kubeadm 업그레이드/etcd 백업 시나리오는 제한적(컨테이너 노드라).

### minikube
- 👍 입문 친화적, addon 풍부(ingress, metrics-server 등 한 줄로).
- 👎 기본 단일 노드 지향. 가볍게 개념 확인용.

### killercoda.com
- 👍 **무료**, 브라우저에서 즉시 실 클러스터, CKA/CKAD 시나리오 다수. 시험과 UX 유사, 설치 부담 0.
- → 이동 중 / 짬짬이 문제풀이에 최적.

### kodekloud.com
- 👍 CKA 강의 + 브라우저 랩이 잘 짜여 있음. Mumshad의 CKA 코스가 사실상 표준 교재.
- 👎 유료(구독).

---

## 추천 셋업 (Mac 기준)

### A. 로컬 메인 — kind 멀티노드

```bash
brew install kind kubectl
# colima로 도커 런타임만 제공 (내장 k3s는 끄고 — 최초 1회만 사양 플래그)
colima start --cpu 4 --memory 8 --disk 60 --kubernetes=false

cat <<EOF | kind create cluster --name study --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF
```

일상 매니페스트 / 스케줄링 / RBAC / NetworkPolicy 실습은 여기서.

### B. CKA 핵심(kubeadm) — 두 갈래로 나눠 연습

kubeadm 관련 작업은 "이미 깔린 클러스터를 운영하는 것"과 "맨바닥부터 세우는 것"으로 나뉜다. 각각 최적 환경이 다르다.

**B-1. 업그레이드 / etcd 백업·복구 숙달 → killercoda (무료, VM 불필요)**

이 둘은 시험 출제 1순위인데, killercoda CKA 트랙이 멀티노드 실 클러스터(etcd가 static pod로 도는 실제 환경)를 브라우저로 제공해서 **굳이 VM을 안 만들어도** 반복 연습이 된다. 10회 이상 손에 익히는 게 목표.
- [killercoda.com/cka](https://killercoda.com/cka) — 시험 스타일 시나리오 25개

**B-2. 클러스터 0 → 1 구축 경험 → Multipass VM 2대 (한 번)**

`kubeadm init` → containerd 설정 → CNI 설치 → join 토큰으로 노드 추가까지 전체 라이프사이클은 직접 세워봐야 원리가 박힌다. 시험에 init부터는 거의 안 나오지만 이해도 측면에서 1회 권장.

```bash
brew install multipass
multipass launch --name cp     --cpus 2 --memory 2G 22.04
multipass launch --name node01 --cpus 2 --memory 2G 22.04
# 각 VM에서 kubeadm init / join, containerd 설정, CNI 설치
```

### C. 문제풀이 — killercoda(상시 무료) + killer.sh(시험 등록 후)

### D. 실무 — EKS

CKA 합격 후 / 병행으로, 업무 환경 그대로.

```bash
brew install eksctl awscli
eksctl create cluster --name dev --nodes 2 --node-type t3.medium
```

> ⚠️ EKS는 control plane + EC2 비용이 시간당 과금된다. 실습 끝나면 `eksctl delete cluster` 잊지 말 것.
> ⚠️ CKA 학습 자체는 EKS에서 하지 말 것 — 관리형이라 control plane 내부(etcd/kubeadm)를 만질 수 없어 시험 범위 연습이 불가능하다.

---

## 참고 자료

- [CKA – Linux Foundation Training](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)
- [killer.sh CKA Simulator](https://killer.sh/cka)
- [CKA Exam Report 2026 (post-2025 revision) – DEV](https://dev.to/suzuki0430/cka-certified-kubernetes-administrator-exam-report-2026-dont-rely-on-old-guides-mastering-the-534m)
- [CKA Certification Guide 2026 (v1.35 syllabus) – GitHub](https://github.com/theplatformlab/CKA-Certified-Kubernetes-Administrator)
- [killercoda](https://killercoda.com/) · [KodeKloud](https://kodekloud.com/)
