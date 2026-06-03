# 08. CKA 준비

시험 자체에 대한 메모 — 환경/도구/팁/기출 패턴 정리.

## 시험 개요 (2025 개정 기준)

- **형식**: performance-based (실제 터미널에서 작업), 원격 감독
- **시간**: 2시간, 합격선 약 66%
- **버전**: 최신 k8s보다 1~2 버전 낮은 버전으로 진행
- **포함 혜택**: [killer.sh](https://killer.sh/cka) 시뮬레이터 2회 (각 17문항, 36시간 액세스) — 실제보다 어려움
- **재응시**: 1회 무료 재시험 포함(보통)

## 준비 체크리스트

- [ ] kubectl alias/단축키 셋업 (`alias k=kubectl`, 자동완성)
- [ ] `--dry-run=client -o yaml`로 매니페스트 빠르게 생성하는 습관
- [ ] `kubectl explain`으로 필드 찾기
- [ ] 공식 문서 북마크 — 시험 중 kubernetes.io 열람 허용
- [ ] imperative 명령 숙달 (run/create/expose/scale/set)
- [ ] 컨텍스트 전환 (`kubectl config use-context`) — 문제마다 클러스터 다름
- [ ] killer.sh 2세션 풀고 오답 복습
- [ ] killercoda CKA 시나리오 반복

## 자주 쓰는 단축 셋업

```bash
alias k=kubectl
export do="--dry-run=client -o yaml"   # k create deploy nginx --image=nginx $do
export now="--force --grace-period=0"  # 빠른 삭제
source <(kubectl completion bash)      # zsh면 completion zsh
complete -F __start_kubectl k
```

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
