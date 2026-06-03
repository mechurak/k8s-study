# 02. 핵심 개념 — 실습 가이드

> 이 문서는 **직접 따라 하며 실행**하는 실습 가이드다. 명령을 본인이 치고, 결과를 관찰하고, 배운 점은 [README의 정리 섹션](./README.md#정리)에 본인 언어로 적는다.
> 각 단계의 🔎는 **관찰 포인트**, 🧪는 **스스로 해볼 과제**다.

## 사전 준비

```bash
colima start          # 환경 시작 (VM 처음 만들 때만 사양 플래그 필요)
kubectl get nodes     # 노드가 Ready인지 확인
cd 02_core-concepts
```

---

## 실습 1 — Pod 하나 띄우고 라이프사이클 관찰

```bash
# 1) Pod 생성
kubectl apply -f manifests/pod.yaml

# 2) 상태를 여러 번 조회해본다
kubectl get pod nginx -o wide
```
🔎 처음엔 `STATUS`가 `ContainerCreating`이다가 잠시 후 `Running`으로 바뀐다. **왜 바로 Running이 아닐까?** (힌트: 이미지를 받아야 한다)

```bash
# 3) 라이프사이클을 이벤트로 추적
kubectl describe pod nginx        # 출력 맨 아래 Events 섹션을 본다
```
🔎 Events에 찍히는 순서를 확인: `Scheduled → Pulling → Pulled → Created → Started`. 각 단계가 무슨 의미인지 생각해본다.

```bash
# 4) 컨테이너 로그 보기
kubectl logs nginx

# 5) Ready 될 때까지 기다리는 방법
kubectl wait --for=condition=Ready pod/nginx --timeout=60s
```

🔎 `-o wide`로 본 Pod의 **IP**와 **배정된 노드(NODE)**는 무엇인가?
🧪 `kubectl get pod nginx -o yaml | less` 로 실제 오브젝트 전체 구조를 살펴보자 — `apiVersion/kind/metadata/spec/status`가 어떻게 들어있나?

---

## 실습 2 — Deployment & 자기복구(self-healing)

```bash
# 1) Deployment 생성 (Pod 3개를 관리)
kubectl apply -f manifests/deployment.yaml

# 2) 계층 구조를 한눈에
kubectl get deploy,rs,pod -l app=nginx
```
🔎 `Deployment` → `ReplicaSet`(이름에 해시) → `Pod`(이름에 해시-해시) 계층이 보인다. Deployment는 Pod를 **직접** 만드는가, ReplicaSet을 **통해** 만드는가?
🔎 실습 1의 독립 Pod `nginx`도 같은 `app=nginx` 라벨이라 목록에 같이 보일 수 있다. Deployment가 만든 Pod와 이름이 어떻게 다른지 비교해본다.

```bash
# 3) [자기복구 실험] Deployment Pod 하나를 골라 삭제
kubectl get pod -l app=nginx          # 해시 붙은 이름 하나 복사
kubectl delete pod <위에서-고른-이름>

# 4) 즉시 다시 조회
kubectl get pod -l app=nginx
```
🔎 방금 지웠는데 Pod 개수는 몇 개인가? 새로 생긴 Pod(AGE가 짧은 것)가 보이는가? **누가** 다시 만들었을까?

```bash
# 5) [대조 실험] 컨트롤러 유무 차이 — ownerReferences 비교
kubectl get pod nginx -o jsonpath='{.metadata.ownerReferences}'; echo
kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.ownerReferences}'; echo

# 6) 독립 Pod를 지워본다
kubectl delete pod nginx
kubectl get pod -l app=nginx
```
🔎 독립 Pod(`nginx`)의 `ownerReferences`는 비어있고, Deployment Pod는 `ReplicaSet/...`가 들어있다.
🔎 독립 Pod는 지우면 다시 살아나는가? Deployment Pod와 왜 다른가?
🧪 이 차이로부터 "실무에선 왜 Pod를 직접 만들지 않고 Deployment를 쓰는가"를 한 문장으로 정리해보자.

---

## 실습 3 — 스케일링

```bash
kubectl scale deploy/nginx --replicas=5     # 늘리기
kubectl get pod -l app=nginx                # 5개 되는지
kubectl scale deploy/nginx --replicas=3     # 줄이기
kubectl get deploy nginx                    # READY 3/3 확인
```
🔎 `replicas` 숫자만 바꿨는데 Pod 수가 따라 변한다. 이 일을 실제로 수행하는 건 누구(어떤 오브젝트)일까?

---

## 마무리

```bash
# 실습 리소스 정리
kubectl delete -f manifests/ --ignore-not-found

# 또는 환경만 끄고 클러스터는 보존
colima stop
```

## 정리

핵심 개념 정리는 **[README의 정리 섹션](./README.md#정리)** 에 있다(개념 설명 + 명령 요약). 위 실습에서 직접 관찰한 것과 대조하며 체득하자.

## 다음 실습 거리

- Deployment **롤아웃/롤백**: 이미지 버전 변경(`kubectl set image ...`) → `kubectl rollout status/history/undo`
- **probe**(liveness/readiness), **Namespace**, `--dry-run=client -o yaml`로 매니페스트 생성하기
