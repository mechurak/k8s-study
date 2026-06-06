# 02. 핵심 개념 — 실습 ② (운영: dry-run·Namespace·롤아웃)

> 실습 ①([practice1.md](./practice1.md))에서 Pod·Deployment 기본을 다뤘다. 여기선 **CKA에서 매 문제 쓰는 운영 기술**을 손에 익힌다.
> 각 단계의 🔎는 **관찰 포인트**, 🧪는 **스스로 해볼 과제**, 📖는 관련 개념 문서다.

## 사전 준비

```bash
kubectl get nodes          # 노드가 Ready인지
cd 02_core-concepts
```

---

## 실습 1 — dry-run으로 매니페스트 빠르게 만들기 ⭐

CKA는 시간 싸움이라, YAML을 손으로 다 치지 않고 **imperative 명령으로 뼈대를 뽑아** 고쳐 쓴다. 가장 자주 쓰는 기술.

```bash
# 1) 서버에 적용하지 않고 YAML만 출력 (--dry-run=client = "보내지 말고 만들어만 봐")
kubectl create deployment web --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml

# 2) 파일로 저장 → 편집 → 적용
kubectl create deployment web --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > web.yaml
#   (web.yaml을 열어 필요한 부분 수정)
kubectl apply -f web.yaml
kubectl get deploy web
```
🔎 `--dry-run=client`는 **apiserver에 안 보내고** 클라이언트가 YAML만 만든다. `> web.yaml`로 저장해 두면 이후 수정·재적용·버전관리가 쉽다(declarative로 전환).
💡 매번 길게 치기 귀찮으면 단축 변수: `export do='--dry-run=client -o yaml'` → `kubectl create deploy web --image=nginx:1.27 $do > web.yaml`. (시험장 세팅 → [`08_cka-prep`](../08_cka-prep/))

🧪 다른 리소스도 같은 방식으로 뽑아보자 — 무엇이 나오는지 비교:
```bash
kubectl run pod1 --image=nginx:1.27 --dry-run=client -o yaml        # Pod
kubectl expose deployment web --port=80 --dry-run=client -o yaml    # Service
```
🧪 필드를 모를 땐 `kubectl explain deployment.spec.strategy` 로 확인.

📖 개념: [kubectl-basics.md](./kubectl-basics.md) (Imperative vs Declarative, `--dry-run`, `explain`)

---

## 실습 2 — Namespace에서 격리해 작업하기

CKA는 **문제마다 네임스페이스가 다르다.** `-n`을 빼먹으면 엉뚱한 곳을 보게 된다. 격리해서 작업하고 통째로 정리하는 흐름을 익힌다.

```bash
# 1) 전용 네임스페이스 생성
kubectl create namespace core-lab

# 2) 같은 web을 core-lab에도 배포 (default의 web과 '이름이 같아도' 충돌 안 함)
kubectl apply -f web.yaml -n core-lab
kubectl get deploy -n core-lab          # core-lab에 web 있음
kubectl get deploy -n default           # default에도 web 있음 — 서로 다른 칸막이
```
🔎 이름이 같은 `web`이 두 네임스페이스에 **공존**한다. 네임스페이스가 **이름 충돌을 막는 칸막이**임을 직접 본 것.

```bash
# 3) 기본 네임스페이스를 바꿔 -n 생략하기 (자주 쓰는 편의)
kubectl config set-context --current --namespace=core-lab
kubectl get pods                        # 이제 core-lab 기준으로 조회됨

# 4) ⚠️ 끝나면 반드시 default로 되돌린다 (안 그러면 계속 core-lab을 본다)
kubectl config set-context --current --namespace=default
```
🔎 cluster-scoped vs namespaced 확인:
```bash
kubectl get ns                          # Namespace는 클러스터 전역 리소스
kubectl api-resources --namespaced=false | head   # Node/PV/Namespace 등은 ns에 안 속함
```
🧪 `kubectl get pods -A` 로 전체 네임스페이스를 한 번에 보면, `kube-system` 등 시스템 파드까지 보인다.

📖 개념: [labels-namespaces.md](./labels-namespaces.md)

---

## 실습 3 — 롤아웃 / 롤백

이미지 버전을 올리면 Deployment가 **새 ReplicaSet으로 점진 교체**(RollingUpdate)한다. 문제가 생기면 **이전 리비전으로 롤백**한다. CKA·실무 단골.

```bash
# 0) 현재 이미지 확인 — 사람이 볼 땐 -o wide가 가장 빠르다 (IMAGES 열)
kubectl get deploy web -o wide
#   값 하나만 정확히 뽑을 때(스크립트/자동화)는 jsonpath:
kubectl get deploy web -o jsonpath='{.spec.template.spec.containers[0].image}'; echo

# 1) 이미지 올리기 → 롤아웃 시작 (컨테이너 이름은 보통 이미지명 'nginx')
kubectl set image deployment/web nginx=nginx:1.28
# (선택) 이번 변경의 '이유'를 기록 → 아래 rollout history의 CHANGE-CAUSE 열에 표시된다
kubectl annotate deployment/web kubernetes.io/change-cause="bump nginx to 1.28" --overwrite

# 2) 진행 상황 / 결과
kubectl rollout status  deployment/web
kubectl get rs -l app=web              # 새 해시 RS가 생기고 옛 RS는 0개로
kubectl rollout history deployment/web # 리비전 이력 (CHANGE-CAUSE 열에 위 메모가 보임)
```
🔎 `get rs`를 보면 **옛 ReplicaSet은 0개, 새 ReplicaSet이 3개**다. 롤아웃 = RS 사이의 점진 전환.
💡 위 두 번째 줄 `annotate`가 왜 있나? → **변경 이유를 기록**하는 용도다(폐기된 `--record`의 대체). `rollout history`의 **CHANGE-CAUSE** 열을 채운다 — 안 해도 롤아웃 자체는 되지만, 이력 가독성을 위해 남긴다. 동작 원리는 [deployments.md](./deployments.md)의 *CHANGE-CAUSE — 변경 이유 기록* 항목 참고.

```bash
# 3) [실패 → 롤백 실험] 없는 이미지로 롤아웃하면 멈춘다
kubectl set image deployment/web nginx=nginx:9.99-doesnotexist
kubectl rollout status deployment/web --timeout=30s    # 안 끝나고 타임아웃
kubectl get pod -l app=web                             # 새 Pod가 ImagePullBackOff

# 4) 직전(정상) 리비전으로 롤백
kubectl rollout undo deployment/web
kubectl rollout status deployment/web                  # 곧 정상 복구
```
🔎 실패한 롤아웃 중에도 **옛 Pod는 살아있어** 서비스가 안 끊긴다(RollingUpdate의 `maxUnavailable` 덕). 그래서 안전하게 `undo` 가능.
🔎 지금 `kubectl rollout history deployment/web`를 보면 CHANGE-CAUSE가 여러 리비전에서 **똑같이** 보이고 번호가 **건너뛸** 수 있다(예: 1, 3, 4). change-cause는 갱신 안 하면 직전 값이 복사되고, `undo`는 옛 리비전을 재승격하며 번호를 새로 받기 때문 — **정상이다.** "무슨 일이었는지"는 아래처럼 실제 템플릿을 봐야 정확하다:
```bash
kubectl rollout history deployment/web --revision=3   # 그 리비전의 이미지 등 — 깨진 9.99가 보일 것
```
🧪 `kubectl rollout history deployment/web --revision=<있는 번호>` 로 특정 리비전 상세 보기, `kubectl rollout undo deployment/web --to-revision=<번호>` 로 지정 리비전 롤백. (원리 → [deployments.md](./deployments.md))

📖 개념: [deployments.md](./deployments.md) (롤아웃/롤백, 업데이트 전략)

---

## (선택) 실습 4 — probe: liveness / readiness

kubelet이 컨테이너 건강을 점검한다. **liveness 실패 → 재시작**, **readiness 실패 → 트래픽 제외**. 일부러 실패시키며 관찰해보자.

```bash
# 30초 뒤 헬스 파일을 지워 liveness를 일부러 실패시키는 데모 Pod
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      args: ["/bin/sh", "-c", "touch /tmp/healthy; sleep 30; rm /tmp/healthy; sleep 600"]
      livenessProbe:
        exec:
          command: ["cat", "/tmp/healthy"]
        initialDelaySeconds: 5
        periodSeconds: 5
EOF
```
```bash
kubectl get pod liveness-demo -w      # 30~40초 뒤 RESTARTS가 늘어난다 (Ctrl+C로 중단)
kubectl describe pod liveness-demo     # Events에 "Liveness probe failed" + "Killing"
```
🔎 헬스 파일이 사라지자 liveness가 실패 → kubelet이 **컨테이너를 재시작**(RESTARTS 증가). readiness였다면 재시작 대신 **엔드포인트에서 제외**됐을 것(→ Service, [`04`](../04_services-networking/)).

📖 개념: [pods.md](./pods.md#probe-헬스-체크) · [공식: Liveness/Readiness/Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

---

## (보너스) 클러스터 구조 둘러보기

개념 문서로 본 control plane/노드를 실제로 들여다본다.

```bash
kubectl cluster-info                  # apiserver/coredns 엔드포인트
kubectl get nodes -o wide             # 노드 목록 + IP/버전/런타임
kubectl get pods -n kube-system       # coredns, kube-proxy, kindnet 등 시스템 컴포넌트
```
🔎 `kube-system`의 파드들이 클러스터를 떠받친다. (kind에선 etcd/apiserver가 control-plane **컨테이너 안**에서 돌아 목록에 다 안 보일 수 있다.)

📖 개념: [cluster-architecture.md](./cluster-architecture.md)

---

## 마무리

```bash
# 이 실습에서 만든 리소스 정리 (멱등 — 이미 없어도 에러 없음)
kubectl delete deployment web --ignore-not-found      # default의 web
kubectl delete namespace core-lab --ignore-not-found  # core-lab의 web까지 통째로
kubectl delete pod liveness-demo --ignore-not-found
rm -f web.yaml

# 혹시 기본 네임스페이스가 core-lab로 남아있다면 default로
kubectl config set-context --current --namespace=default
```
🔎 **네임스페이스를 지우면 그 안의 모든 리소스가 함께 삭제된다** — 실습 정리에 가장 깔끔한 방법. 단, 이건 **`core-lab`처럼 내가 만든 ns에만** 해당한다. ⚠️ **`default`·`kube-system` 등 시스템(예약) 네임스페이스는 절대 지우지 말 것**(`default`엔 `kubernetes` Service·기본 ServiceAccount가 얽혀 있어 망가진다). `default`를 정리할 땐 위처럼 **리소스만 골라 삭제**한다.

🔎 **정리 확인**: `kubectl get all` 에 `service/kubernetes` **하나만** 남으면 default는 깨끗하다 — 그건 apiserver로 가는 **상시 기본 서비스**라 지우는 게 아니다(이게 보이면 현재 ns가 default로 돌아온 것이기도 함). `kubectl get ns` 로 `core-lab`이 사라졌는지도 확인.
> ⚠️ `kubectl get all`은 이름과 달리 **모든 종류를 보여주진 않는다**(주요 워크로드/서비스류만 — ConfigMap·Secret·PVC·Ingress·ServiceAccount 등은 빠짐). 특정 종류는 `kubectl get configmap,secret,pvc`처럼 따로 확인한다.

> 환경(런타임/클러스터)을 끄는 방법은 [`01_lab-environment`](../01_lab-environment/) 참고.

## 다음 — 03으로

여기까지 하면 02의 핵심은 손에 익은 셈이다. 다음은 [`03_workloads-scheduling`](../03_workloads-scheduling/) — ConfigMap/Secret, 리소스 요청/제한, 스케줄링(노드 선택·taint), 오토스케일링(HPA). probe·롤아웃에서 본 워크로드 운영이 이어진다.
