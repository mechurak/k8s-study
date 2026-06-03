# 06. 클러스터 구축과 운영

> CKA 도메인: **Cluster Architecture, Installation & Configuration (~25%)**

CKA의 핵심 구간. **kind로는 부족**하니 환경을 나눠서 실습한다 (`01_lab-environment/README.md` 참고):
> - 업그레이드 / etcd 백업·복구 반복 → **killercoda** (무료 멀티노드)
> - `kubeadm init`부터 클러스터 구축 1회 → **Multipass VM**

## 학습 목표

- [ ] 클러스터 아키텍처 — control plane 컴포넌트(apiserver, etcd, scheduler, controller-manager), kubelet, kube-proxy
- [ ] 인프라 준비 — 노드 OS/커널 설정, 컨테이너 런타임(containerd) 사전 준비
- [ ] **kubeadm** — `init` / `join`, 토큰, CNI 설치
- [ ] **클러스터 업그레이드** — `kubeadm upgrade plan/apply`, kubelet/kubectl 업그레이드, 노드 drain/uncordon
- [ ] **etcd 백업·복구** — `etcdctl snapshot save/restore` (static pod 구조 이해)
- [ ] **HA(고가용성) 컨트롤 플레인** — 다중 control plane, stacked vs external etcd, 앞단 로드밸런서
- [ ] **RBAC** — Role/ClusterRole, RoleBinding/ClusterRoleBinding, ServiceAccount
- [ ] **확장 인터페이스 개념** — CNI(네트워크) / CSI(스토리지) / CRI(런타임)
- [ ] Static Pod, manifests 디렉터리
- [ ] **CRD / 오퍼레이터** — CRD 설치, 오퍼레이터 설치·설정
- [ ] **Kustomize** — base/overlay, `kubectl apply -k`, patch
- [ ] **Helm** — chart 구조, `helm install/upgrade/rollback`, values, repo (컴포넌트 설치에 활용)

## 정리

> (학습하며 채워나갈 자리)

## 참고

- [kubeadm으로 클러스터 구성](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [클러스터 업그레이드](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [etcd 백업/복구](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/) · [Helm](https://helm.sh/docs/)
