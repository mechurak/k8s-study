# ArgoCD 운영 레시피 (자주 하는 작업)

> 처음 한 번 따라 하는 실습은 [practice.md](./practice.md), 개념은 [argocd.md](./argocd.md). 이 문서는 **개발·운영하며 반복적으로 꺼내 쓰는 작업** 모음이다. 필요할 때마다 항목을 쌓아 간다.

## 공통 주의 — ArgoCD가 관리하는 리소스를 직접 손볼 때

ArgoCD Application이 `selfHeal`(자동 교정)로 떠 있으면, 내가 `kubectl`로 리소스를 지우거나 바꿔도 **ArgoCD가 Git 상태로 즉시 되돌린다.** 그래서 수동 작업이 "먹히지 않는" 것처럼 보인다. 직접 손봐야 할 땐 보통 둘 중 하나:

- **잠깐 자동 sync를 끈다**: `argocd app set <app> --sync-policy none` → 작업 → `--sync-policy automated`로 복원
- 또는 **Git을 고쳐서** ArgoCD가 그 변경을 정상 경로로 반영하게 한다(원칙적으로 이게 GitOps 정석).

---

## 개발 중 PostgreSQL DB를 처음부터 다시 만들기 (데이터 초기화)

> 개발 단계라 데이터를 날려도 될 때. **운영에서는 절대 이렇게 하지 않는다.**

**핵심**: **PVC를 지워야 데이터가 사라진다.** Pod·StatefulSet·Application을 지워도 `volumeClaimTemplates`로 만들어진 PVC는 **데이터 보호를 위해 남는다**(→ [05_storage](../05_storage/)). 즉 "DB 초기화 = PVC 명시적 삭제".

```bash
NS=database        # 환경에 맞게 (로컬 실습이면 database / PVC는 data-postgres-0)

# 1) ArgoCD가 잠깐 손 떼게 (selfHeal이 replicas를 되돌리는 걸 막음)
argocd app set postgres --sync-policy none
#    UI: App → APP DETAILS → DISABLE AUTO-SYNC

# 2) StatefulSet + PVC 삭제  ← PVC 삭제가 '데이터 날리는' 지점
kubectl -n $NS delete statefulset postgres
kubectl -n $NS delete pvc data-postgres-0
kubectl -n $NS get pvc          # 사라졌는지 확인

# 3) 다시 ArgoCD에 맡기기 → Git대로 재배포, 빈 DB로 initdb
argocd app set postgres --sync-policy automated
#    (또는 한 번만 동기화: argocd app sync postgres)
```
> `selfHeal`이 **꺼져 있다면** 1·3단계 없이 `delete statefulset` + `delete pvc` 후 `argocd app sync postgres`만 하면 된다.

**대안 — Application째 갈아엎기** (이때도 PVC는 따로 지워야 함):
```bash
argocd app delete postgres --cascade            # SS/Svc/Secret 정리 (PVC는 남음)
kubectl -n $NS delete pvc data-postgres-0        # ← 데이터 삭제
kubectl apply -f manifests/argocd-apps/postgres-app.yaml   # 새로 배포
```

**DB를 비웠으면 backend도 재시작** — 시작 시 마이그레이션을 돌리는 구조라면 스키마를 다시 잡도록:
```bash
kubectl -n <backend-ns> rollout restart deploy/backend
```

**확인**:
```bash
kubectl -n $NS get pvc,pod                                          # 새 PVC, postgres-0 Running
kubectl -n $NS exec -it postgres-0 -- psql -U app -d appdb -c '\dt' # 테이블이 비어있으면 성공
```

---

## 참고

- [argocd.md](./argocd.md) — 개념(Sync 정책, OutOfSync 진단)
- [05_storage](../05_storage/) — PVC/StorageClass 라이프사이클(왜 PVC는 안 지워지나)
