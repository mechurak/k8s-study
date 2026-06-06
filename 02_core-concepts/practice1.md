# 02. 핵심 개념 — 실습 ① (Pod·Deployment 기본)

> 이 문서는 **직접 따라 하며 실행**하는 실습 가이드다. 명령을 본인이 치고, 결과를 관찰하며 손에 익힌다. 개념 설명은 [README의 개념 문서들](./README.md#다루는-내용)에서 확인한다.
> 각 단계의 🔎는 **관찰 포인트**, 🧪는 **스스로 해볼 과제**다.
> 이어지는 운영 심화(dry-run·Namespace·롤아웃/롤백)는 **[practice2.md](./practice2.md)** 에 있다.

## 사전 준비

클러스터가 떠 있어야 한다. 실습 환경(colima·kind 등) 준비/시작은 [`01_lab-environment`](../01_lab-environment/) 참고.

```bash
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

## 실습 2 — Pod에 접속해서 직접 확인 (exec · port-forward)

실습 1에서 띄운 `nginx` Pod가 **진짜 도는지** 두 방향에서 만져본다: **안으로 들어가기**(exec)와 **밖에서 접속하기**(port-forward).

### (가) 컨테이너 안으로 — `kubectl exec`

```bash
kubectl exec -it nginx -- bash      # 컨테이너 안 셸로 진입 (bash 없으면 -- sh)
# ↓ 아래는 '컨테이너 안'에서 실행
curl -s localhost | head            # nginx가 자기 자신에게 응답하나
exit                                # 빠져나오기
```
🔎 진입하면 프롬프트가 바뀐다(컨테이너 안). 여기서 `localhost`는 이제 **그 컨테이너 자신**이다. `-it`는 `-i`(stdin 유지)+`-t`(tty)로, 대화형 셸에 필요.
💡 셸 없이 **명령 하나만** 실행도 된다: `kubectl exec nginx -- nginx -v` (`--` 뒤가 컨테이너에서 돌 명령).

### (나) 내 노트북에서 접속 — `kubectl port-forward`

```bash
kubectl port-forward pod/nginx 8080:80   # 로컬 8080 → Pod의 80 (터널, 이 터미널은 포그라운드로 점유됨)
```
**다른 터미널**을 열어서:
```bash
curl -s localhost:8080 | head            # "Welcome to nginx!" 가 보이면 성공
```
브라우저로 http://localhost:8080 을 열면 **nginx 환영 페이지**가 뜬다. 끝내려면 port-forward 터미널에서 `Ctrl+C`.

🔎 실습 1의 `-o wide`로 본 **Pod IP(예: `10.244.x.x`)로 브라우저에서 바로 못 들어가는 이유**는?
그 IP는 **클러스터 내부 네트워크**다. kind 클러스터는 도커 컨테이너 안에서 돌기 때문에 macOS 호스트에선 그 IP에 직접 닿지 못한다. `port-forward`는 apiserver를 거쳐 뚫는 **임시 터널**일 뿐. → "외부에서 안정적으로 접근"하려면 결국 **Service/Ingress**가 필요하다(→ [`04_services-networking`](../04_services-networking/)).

> ⚠️ **port-forward는 "서비스 노출"이 아니다.** 클러스터엔 아무것도 안 만들고, **kubectl을 켠 내 PC에서 나만**, 켜둔 동안만 쓰는 **개인 디버그 터널**이다(apiserver 경유, Pod 1개로). 반면 **NodePort**는 실제 Service 오브젝트라 **모든 노드의 포트**를 열어 누구나 접속·로드밸런싱되고 영속한다. 외부 노출 사다리(ClusterIP→NodePort→LoadBalancer→Ingress)는 [`04_services-networking`](../04_services-networking/)에서 다룬다.

🧪 (대조 실험) 클러스터 **안**에선 Pod IP로 바로 될까? 임시 파드를 띄워 확인:
```bash
IP=$(kubectl get pod nginx -o jsonpath='{.status.podIP}')   # Pod IP를 변수에
kubectl run tmp --rm -i --restart=Never --image=busybox -- wget -qO- "$IP" | head
# --rm: 끝나면 자동 삭제 / 클러스터 안에서는 Pod IP로 바로 응답이 온다
```
🔎 같은 IP인데 **안에선 되고 밖(호스트)에선 안 된다** — Pod IP가 "내부 전용"임을 직접 확인하는 셈.

### exec vs port-forward vs debug — 언제 뭘?

| 하고 싶은 것 | 명령 | 메모 |
|---|---|---|
| 컨테이너 **안에서** 명령/셸 | `kubectl exec -it <pod> -- sh` | 가장 흔함. 안에서 둘러보기·로그 위치 확인 등 |
| 밖에서 그 **포트에 접속** | `kubectl port-forward <pod> 8080:80` | 빠른 단발 접근(개발·디버깅). Service 없이도 가능 |
| 셸 **없는** 이미지·**노드** 디버깅 | `kubectl debug ...` | distroless 등 `exec`가 안 될 때. 임시 디버그 컨테이너 부착 → [`07_troubleshooting`](../07_troubleshooting/) |

> `debug`는 nginx처럼 셸 있는 이미지엔 필요 없다. **셸 없는 최소 이미지**(distroless)나 **노드 자체**를 들여다볼 때 쓰는 한 단계 위 도구라, 지금은 "이런 게 있다"만 알아두면 된다.

---

## 실습 3 — Deployment & 자기복구(self-healing)

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
# (가) 독립 Pod nginx → 소유자 없음 → []
kubectl get pod nginx -o jsonpath='{.metadata.ownerReferences}'; echo
# (나) Deployment가 만든 Pod(위 목록의 해시 붙은 이름) 하나 → 소유자 = ReplicaSet
kubectl get pod <해시-붙은-Pod-이름> -o jsonpath='{.metadata.ownerReferences}'; echo
```
🔎 독립 Pod(`nginx`)의 `ownerReferences`는 비어있고, Deployment Pod는 `ReplicaSet/...`가 들어있다.
💡 `kubectl describe pod <이름>`의 **`Controlled By`** 줄에서도 같은 정보를 더 읽기 쉽게 볼 수 있다(Deployment Pod엔 `ReplicaSet/...`, 독립 Pod엔 그 줄이 없음).

```bash
# 6) 독립 Pod를 지워본다 (컨트롤러가 없으니 안 살아남)
kubectl delete pod nginx
kubectl get pod -l app=nginx
```
🔎 독립 Pod는 지우면 다시 살아나는가? Deployment Pod와 왜 다른가?
🧪 이 차이로부터 "실무에선 왜 Pod를 직접 만들지 않고 Deployment를 쓰는가"를 한 문장으로 정리해보자.

---

## 실습 4 — 스케일링

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
# 실습 리소스 정리 — manifests/ 폴더의 모든 yaml이 정의한 리소스를 삭제
# (-f 는 폴더도 받음 / --ignore-not-found: 이미 없는 건 에러 없이 넘어감 = 멱등)
kubectl delete -f manifests/ --ignore-not-found
```

> 환경(런타임/클러스터)을 끄는 방법은 [`01_lab-environment`](../01_lab-environment/) 참고. 보통은 리소스만 정리하면 충분하다.

## 개념 정리는?

이 실습에서 다룬 개념(Pod·라이프사이클·ReplicaSet/Deployment·자기복구·스케일링)은 **[개념 문서들](./README.md#다루는-내용)** 에 정리돼 있다. 위에서 직접 관찰한 것과 대조하며 체득하자.

## 다음 — 실습 ②로

CKA에서 매 문제 쓰는 운영 기술은 **[practice2.md](./practice2.md)** 에서 이어 한다:

- ⭐ `--dry-run=client -o yaml`로 **매니페스트 빠르게 만들기** (imperative → yaml)
- **Namespace** — 격리해서 작업하고 통째로 정리
- Deployment **롤아웃/롤백** (`set image` → `rollout status/history/undo`)
- (선택) **probe**(liveness/readiness), 클러스터 구조 둘러보기
