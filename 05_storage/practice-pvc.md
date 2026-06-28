# Storage 집중 실습 — PVC·PV·StorageClass·StatefulSet (CKA)

> 목표: CKA Storage(~10%)·Workloads(~15%)에서 바로 점수 되는 **순수 메커니즘**을 손에 익힌다.
> 앱은 `busybox`로 단순화 — ClickHouse 같은 실전 응용은 [`12_data-stores/practice-clickhouse.md`](../12_data-stores/practice-clickhouse.md).
>
> 개념은 [README](./README.md) 정리 참고. 클러스터 준비는 [`01_lab-environment/kind.md`](../01_lab-environment/kind.md).

전제: kind 클러스터가 떠 있고 `kubectl`이 그 클러스터를 가리킨다.

```bash
kubectl cluster-info
kubectl get storageclass        # (default) 표시된 SC 확인 — kind는 standard(local-path)
```

> 🔎 **관찰 포인트**: 기본 SC `standard`의 `VOLUMEBINDINGMODE`가 **`WaitForFirstConsumer`** 다. → PVC를 만들어도 **파드가 쓰기 전까진 PV가 안 만들어진다**(아래 2절에서 바로 확인). `kubectl get sc standard -o yaml`로 `provisioner`·`reclaimPolicy`도 보자.

실습용 네임스페이스:

```bash
kubectl create namespace storage
```

> 아래 명령은 `-n storage`를 붙인다. 귀찮으면 `kubectl config set-context --current --namespace=storage`로 기본 네임스페이스를 바꿔도 된다(시험에서도 유용).

---

## 1. emptyDir — 파드와 운명을 같이하는 임시 볼륨

먼저 "영속이 아닌" 볼륨부터. 한 파드 안 두 컨테이너가 디렉터리를 공유한다.

```bash
kubectl -n storage run ed --image=busybox:1.36 --restart=Never \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "app", "image": "busybox:1.36",
      "command": ["sh","-c","echo hello > /cache/a.txt && sleep 3600"],
      "volumeMounts": [{"name":"c","mountPath":"/cache"}]
    }],
    "volumes": [{"name":"c","emptyDir":{}}]
  }
}' -- sh
```

```bash
kubectl -n storage exec ed -- cat /cache/a.txt        # hello
kubectl -n storage delete pod ed                      # 파드 삭제 → /cache 데이터도 소멸
```

> 🔎 emptyDir는 **파드가 사라지면 같이 사라진다**. 영속이 필요하면 아래 PVC로 간다.

---

## 2. 동적 프로비저닝 — PVC만 만들면 PV가 자동 생성

가장 흔한 실전 패턴. StorageClass가 PVC 요청을 받아 PV를 즉석에서 만든다.

```bash
kubectl -n storage apply -f manifests/pvc-dynamic.yaml
kubectl -n storage get pvc                            # STATUS 확인
```

> 🔎 **관찰 포인트**: STATUS가 `Bound`가 아니라 **`Pending`**! 1절에서 본 `WaitForFirstConsumer` 때문이다 — 아직 이 PVC를 **쓰는 파드가 없어서** PV 생성을 미룬 것. `kubectl -n storage describe pvc pvc-demo`의 Events에 `waiting for first consumer` 가 보인다.

이제 PVC를 쓰는 파드를 띄운다:

```bash
kubectl -n storage apply -f manifests/pod-with-pvc.yaml
kubectl -n storage get pvc,pv                         # 파드가 뜨자 PVC가 Bound, PV 자동 생성
```

데이터를 쓰고, **파드를 지웠다 다시 만들어** 영속을 확인한다:

```bash
kubectl -n storage exec pvc-pod -- sh -c 'echo "persist me" > /data/note.txt'
kubectl -n storage delete pod pvc-pod
kubectl -n storage apply -f manifests/pod-with-pvc.yaml      # 같은 PVC에 다시 붙음
kubectl -n storage exec pvc-pod -- cat /data/note.txt        # persist me — 데이터 유지!
```

> 🔎 **핵심**: 파드는 사라졌다 살아나도 **같은 PVC→PV**에 재결합해 데이터가 유지된다. 이게 영속 저장소의 본질.

---

## 3. PVC가 Pending에서 안 넘어갈 때 — 디버깅 (시험 단골)

일부러 **존재하지 않는 StorageClass**를 요청해 본다:

```bash
kubectl -n storage create -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-broken
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
  storageClassName: no-such-class      # 없는 클래스
EOF

kubectl -n storage get pvc pvc-broken                 # Pending
kubectl -n storage describe pvc pvc-broken            # ← Events를 읽는 게 포인트
```

> 🔎 **관찰 포인트**: Events에 `storageclass.storage.k8s.io "no-such-class" not found`. **PVC가 Pending이면 항상 `describe`로 Events부터** 본다. 흔한 원인: SC 이름 오타 / 기본 SC 없음 / 용량·accessMode 불일치 / 맞는 정적 PV 없음.

```bash
kubectl -n storage delete pvc pvc-broken
```

---

## 4. 정적 PV → PVC 바인딩 (관리자가 직접 PV 제공)

동적 SC 없이, 관리자가 **PV를 미리 만들고** PVC가 거기 붙는 시나리오.

```bash
kubectl apply -f manifests/pv-static.yaml             # PV는 클러스터 스코프(-n 불필요)
kubectl -n storage apply -f manifests/pvc-static.yaml
kubectl get pv ; kubectl -n storage get pvc pvc-manual
```

> 🔎 **관찰 포인트**
> - `pvc-manual`이 `pv-manual`과 **`Bound`** (둘 다 `storageClassName: manual`, 용량·accessMode가 맞아서).
> - `storageClassName: manual`은 동적 프로비저너가 없는 임의 이름 → **동적 생성이 아니라 기존 PV 매칭**으로 동작.
> - PV의 `RECLAIM POLICY`가 `Retain` → PVC를 지워도 PV·데이터는 남는다(아래 6절에서 확인).

---

## 5. StatefulSet — volumeClaimTemplates로 파드별 전용 PVC

DB처럼 **파드마다 자기 디스크**가 필요한 경우. (워크로드 개념은 [`03_workloads-scheduling`](../03_workloads-scheduling/), Headless는 [`04_services-networking`](../04_services-networking/#headless-service--파드를-콕-집어-부른다))

```bash
kubectl -n storage apply -f manifests/statefulset.yaml
kubectl -n storage get pods -w                        # web-0 → web-1 순서로 생성(Ctrl+C)
```

```bash
kubectl -n storage get statefulset,pod,svc,pvc
```

> 🔎 **관찰 포인트**
> - 파드 이름이 **`web-0`, `web-1`** (고정 순번. Deployment의 랜덤 해시와 대비).
> - PVC가 **`data-web-0`, `data-web-1`** 로 **파드마다 하나씩** 자동 생성됨 — `volumeClaimTemplates`의 결과.
> - Service `web`의 `CLUSTER-IP`가 **`None`**(Headless).

파드별 데이터 영속 확인 — **파드를 지워도 같은 PVC에 재결합**:

```bash
kubectl -n storage exec web-0 -- sh -c 'echo "I am web-0" > /data/id.txt'
kubectl -n storage delete pod web-0                   # StatefulSet이 web-0을 같은 PVC로 재생성
kubectl -n storage get pods -w                        # web-0 다시 Running(Ctrl+C)
kubectl -n storage exec web-0 -- cat /data/id.txt     # I am web-0 — 유지!
```

> 🔎 파드를 지워도 PVC(`data-web-0`)는 안 지워진다 → 새 `web-0`이 **그 PVC에 다시 붙어** 데이터 유지. StatefulSet이 "안정적 신원 + 전용 디스크"를 보장하는 핵심 동작.

---

## 6. 정리(cleanup) — reclaimPolicy 차이 관찰

```bash
# StatefulSet 삭제해도 PVC는 남는다(의도된 보호)
kubectl -n storage delete statefulset web
kubectl -n storage get pvc                            # data-web-0,1 아직 있음 → 수동 삭제 필요
kubectl -n storage delete pvc --all

# 정적 PV: PVC 지워도 Retain이라 PV는 Released 상태로 남음
kubectl -n storage delete pvc pvc-manual
kubectl get pv pv-manual                              # STATUS: Released (Retain이라 자동삭제 안 됨)
kubectl delete pv pv-manual                           # 수동 정리

# 나머지 한 방에
kubectl delete namespace storage
```

> ⚠️ **StatefulSet을 지워도 PVC는 자동 삭제되지 않는다**(데이터 보호). 직접 지워야 한다. 반대로 `Delete` reclaimPolicy(동적 SC 기본)는 PVC를 지우면 PV·실제 디스크까지 사라진다.

---

## 🧪 과제

1. **accessMode 불일치**: `pvc-static.yaml`의 accessMode를 `ReadWriteMany`로 바꿔 apply → `pv-manual`(RWO)과 바인딩되나? `describe`로 이유 확인.
2. **기본 SC로 정적 흉내**: `pvc-dynamic.yaml`에 `storageClassName: ""`(빈 문자열)를 넣으면? (동적 프로비저닝이 꺼져 맞는 PV가 없으면 Pending) — 빈 문자열과 생략의 차이를 정리해 보자.
3. **PVC 확장 시도**: `kubectl get sc standard -o yaml | grep allowVolumeExpansion` 확인 → kind 기본은 확장 미지원이라 PVC `storage`를 키워도 안 늘 수 있다. 어떤 SC라야 `kubectl edit pvc`로 확장되나? (`allowVolumeExpansion: true`)
4. **순차성 관찰**: StatefulSet `replicas`를 3으로 늘리고 `kubectl get pods -w` → 정말 web-2가 web-1 다음에 뜨나? 스케일 다운하면 삭제 순서는?
5. **다음 단계**: 같은 프리미티브(StatefulSet+PVC+Headless)를 진짜 DB로 → [`12_data-stores/practice-clickhouse.md`](../12_data-stores/practice-clickhouse.md).

---

## 시험 팁 (imperative)

```bash
# YAML 빠르게 뽑아 편집 (처음부터 타이핑 금지)
kubectl run pvc-pod --image=busybox:1.36 --dry-run=client -o yaml -- sleep 3600 > pod.yaml
# PVC·PV·StatefulSet은 imperative 생성 명령이 없음 → 위 manifests/ 를 템플릿으로 복사해 수정
kubectl get pvc,pv,sc -A                       # 빠른 현황 (pvc는 vc로 줄여도 됨)
kubectl describe pvc <name>                     # Pending 디버깅은 항상 여기부터
```
