# 07_troubleshooting

> CKA 도메인: **Troubleshooting (~30%)** — 출제 비중이 가장 크다.

문제 상황을 빠르게 진단하고 고치는 능력. 시험 득점의 핵심이라 **반복 실습** 필수.

## 다루는 내용
- 진단 기본기 — `kubectl describe`, `logs`, `events`, `get -o wide`
- 파드 문제 — Pending / CrashLoopBackOff / ImagePullBackOff / OOMKilled
- 노드 문제 — NotReady, kubelet 상태(`systemctl status kubelet`, `journalctl`)
- 컨트롤 플레인 문제 — apiserver/etcd/scheduler static pod 점검
- 네트워킹 문제 — 서비스/DNS/NetworkPolicy 연결 안 됨
- 권한 문제 — RBAC `kubectl auth can-i`
- 리소스 부족 — requests/limits, 노드 capacity
- **리소스 사용량 모니터링** — `kubectl top` (metrics-server 설치), 노드/파드 사용량
- **컨테이너 출력 스트림** — `kubectl logs`(--previous, -c), stdout/stderr 평가
- (도구) `crictl`로 컨테이너 런타임 레벨 디버깅

## 정리

> (학습하며 채워나갈 자리. 겪은 증상 → 원인 → 해결 패턴을 표로 쌓으면 좋다)

## 참고

- [Troubleshooting Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Troubleshooting Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
