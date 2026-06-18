# 05_storage

> CKA 도메인: **Storage (~10%)**

파드에 영속 저장소를 연결하는 방법.

## 다루는 내용
- Volume 개요 — emptyDir, hostPath
- PersistentVolume (PV) / PersistentVolumeClaim (PVC)
- StorageClass — 동적 프로비저닝
- accessModes (RWO/ROX/RWX), reclaimPolicy
- volumeMode, PVC 확장(resize)
- ConfigMap/Secret를 볼륨으로 마운트 (03번과 연계)
- (실무) EKS의 EBS/EFS CSI 드라이버 → `09_aws-eks`
- **오브젝트 스토리지(S3 호환) — MinIO** (실무, CKA 밖) → [minio.md](./minio.md)
  - PV/PVC(block/file)와 **다른 계층** — 마운트가 아니라 S3 API로 접근. 온프렘 "사내 S3"
  - ⚠️ 2025~2026 오픈소스 상황 변화(콘솔 관리기능 제거·저장소 아카이브)·대안 포함

## 정리

> (학습하며 채워나갈 자리)

## 참고

- [Storage](https://kubernetes.io/docs/concepts/storage/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
