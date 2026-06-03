# 03. 워크로드와 스케줄링

> CKA 도메인: **Workloads & Scheduling (~15%)**

워크로드 컨트롤러와, 파드가 어느 노드에 어떻게 배치되는지(스케줄링).

## 다루는 내용
- Deployment 심화 — 스케일링, 롤아웃 히스토리/롤백
- DaemonSet, StatefulSet 개요
- Job / CronJob
- ConfigMap & Secret — env/volume 주입
- 리소스 관리 — requests/limits, QoS class
- 스케줄링 — nodeSelector, affinity/anti-affinity
- Taint & Toleration
- 수동 스케줄링, `nodeName`
- **워크로드 오토스케일링** — HPA(metrics-server 기반), (개념) VPA / Cluster Autoscaler

## 정리

> (학습하며 채워나갈 자리)

## 참고

- [Workloads](https://kubernetes.io/docs/concepts/workloads/)
- [Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)
