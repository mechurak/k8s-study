# 08. CKA 준비

시험 자체에 대한 메모 — 환경/도구/팁/기출 패턴 정리.

## 시험 개요 (2025 개정 기준)

- **형식**: performance-based (실제 터미널에서 작업), 원격 감독
- **시간**: 2시간, 합격선 약 66%
- **버전**: 최신 k8s보다 1~2 버전 낮은 버전으로 진행
- **포함 혜택**: [killer.sh](https://killer.sh/cka) 시뮬레이터 2회 (각 17문항, 36시간 액세스) — 실제보다 어려움
- **재응시**: 1회 무료 재시험 포함(보통)

## 준비 체크리스트

- kubectl alias/단축키 셋업 (`alias k=kubectl`, 자동완성)
- `--dry-run=client -o yaml`로 매니페스트 빠르게 생성하는 습관
- `kubectl explain`으로 필드 찾기
- 공식 문서 북마크 — 시험 중 kubernetes.io 열람 허용
- imperative 명령 숙달 (run/create/expose/scale/set)
- 컨텍스트 전환 (`kubectl config use-context`) — 문제마다 클러스터 다름
- killer.sh 2세션 풀고 오답 복습
- killercoda CKA 시나리오 반복

## 자주 쓰는 단축 셋업

**시험 들어가면 첫 문제에서 제일 먼저 박는 셋업.** 순서 주의 — completion을 먼저 로드해야 `__start_kubectl`이 정의돼 `k` 완성이 붙는다.

```bash
source <(kubectl completion bash)            # ① __start_kubectl 정의 (zsh면 completion zsh)
alias k=kubectl                              # ② k = kubectl
complete -o default -F __start_kubectl k     # ③ k에도 탭 자동완성 연결 (-o default = 후보 없으면 파일명 완성 폴백)
export do="--dry-run=client -o yaml"         # k create deploy nginx --image=nginx $do
export now="--force --grace-period=0"        # 빠른 삭제: k delete pod x $now
```

| 줄 | 의미 |
|---|---|
| `source <(kubectl completion bash)` | kubectl 자동완성 스크립트 로드 → 완성 함수 `__start_kubectl` 정의됨 |
| `alias k=kubectl` | `k get po` = `kubectl get po` (실행 단축) |
| `complete -o default -F __start_kubectl k` | 별칭 `k`에 **kubectl과 같은 완성 함수**를 연결(별칭은 자동으로 안 붙음) |

### ⚠️ 적용 범위 — 어디까지 유지되나

alias·completion은 **"그 셸 세션"에 산다.** 한 번 세팅하면:

| 상황 | 유지? |
|---|---|
| **같은 터미널에서 다음 문제로** | ✅ 유지. 문제는 `kubectl config use-context …`로 **클러스터만 전환**하지 셸을 새로 안 띄움 |
| **새 터미널 탭/창** | ❌ 거기서 다시 세팅해야 함 |
| **`ssh node01` 로 다른 노드** | ❌ **유지 안 됨** — 별도 머신이라 베이스의 alias가 안 따라감 |

→ 베이스 터미널 한 곳에서 세팅하면 **대부분의 문제에 계속 적용**된다. 더 튼튼하게 하려면 **`~/.bashrc`에 ①~③을 박아** 새 셸에서도 자동 적용:

```bash
cat <<'EOF' >> ~/.bashrc
source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
EOF
source ~/.bashrc
```

> ⚠️ **ssh로 옮겨간 노드는 예외다.** kubelet/static pod 고치는 문제로 `ssh node01` 하면 베이스 alias가 없다 — 그 노드에선 `kubectl` 풀네임을 쓰거나 거기서 다시 alias를 친다. (ssh 노드 작업은 보통 `kubectl` 호출이 적어 풀네임이 더 빠를 때가 많다.)
>
> 💡 최근 CKA 환경엔 `k` alias가 미리 깔린 경우도 있지만, 없을 때를 대비해 **첫 문제에서 ①~③부터 박는 습관**이 안전.

## 기출/빈출 패턴

> (풀어본 문제 유형을 여기에 쌓기)

- etcd 백업/복구
- 클러스터 업그레이드
- RBAC 설정
- NetworkPolicy 작성
- 깨진 노드/컴포넌트 복구

## 참고

- [CKA – Linux Foundation](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)
- [CKA Curriculum (GitHub)](https://github.com/cncf/curriculum)
- [killer.sh](https://killer.sh/cka) · [killercoda CKA](https://killercoda.com/cka)
