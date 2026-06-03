# kubectl 기본 & 매니페스트 구조

> `kubectl`은 apiserver와 대화하는 CLI다. CKA는 **시간 싸움**이라 kubectl을 빠르게 다루는 게 곧 점수다.

## 명령 구조

```
kubectl [command] [type] [name] [flags]
        예: kubectl   get     pod    nginx   -o wide
```

자주 쓰는 command:

| 명령 | 용도 |
|---|---|
| `get` | 목록/요약 조회 (`-o wide`, `-o yaml`, `--show-labels`, `-A`) |
| `describe` | 상세 + **Events**(디버깅 핵심) |
| `logs` | 컨테이너 로그 (`-f` 팔로우, `--previous` 이전 컨테이너, `-c` 컨테이너 지정) |
| `exec` | 컨테이너 안에서 명령 (`-it ... -- sh`) |
| `apply` | 매니페스트로 선언적 적용/갱신 |
| `delete` | 삭제 (`-f file`, `--all`, `--grace-period=0 --force`) |
| `edit` | 실행 중 오브젝트 즉석 수정 |
| `scale` | 레플리카 수 변경 |

> 💡 `kubectl get deploy,rs`처럼 **여러 종류를 한 번에 조회**하면 NAME이 `종류.그룹/이름`(예: `deployment.apps/nginx`)으로 표시된다 — 줄마다 종류를 구분하려는 것. 한 종류만 조회하면 그냥 `nginx`로 나온다.
>
> 💡 `-f`는 파일뿐 아니라 **폴더**도 받는다(`apply`·`delete` 공통 — 폴더 안 모든 매니페스트를 한 번에 처리). `delete`에 `--ignore-not-found`를 붙이면 대상이 이미 없어도 에러 없이 넘어가 **정리가 멱등**해진다.

## Imperative vs Declarative

| | Imperative(명령형) | Declarative(선언형) |
|---|---|---|
| 방식 | "이걸 해라" 명령 | "이 상태가 되게 하라" 파일 |
| 예 | `kubectl run`, `create`, `scale`, `set` | `kubectl apply -f` |
| 장점 | **빠름**(시험에서 유리) | 재현·버전관리·협업 |
| 쓰는 곳 | 시험, 빠른 실험 | 실무(Git에 매니페스트 관리) |

> CKA 전략: **간단한 건 imperative로 빠르게, 복잡한 건 `--dry-run`으로 yaml 뽑아 수정.**

### 자주 쓰는 imperative 명령

```bash
kubectl run nginx --image=nginx                          # Pod 하나
kubectl create deployment web --image=nginx --replicas=3 # Deployment
kubectl expose deployment web --port=80                  # Service
kubectl scale deployment web --replicas=5                # 스케일
kubectl set image deployment/web nginx=nginx:1.28        # 이미지 변경(롤아웃)
kubectl create configmap cm1 --from-literal=KEY=val      # ConfigMap
```

## ⭐ 매니페스트를 빠르게 만들기 — `--dry-run=client -o yaml`

서버에 적용하지 않고 **YAML만 생성**해준다. 시험 필수 기술.

```bash
# 실행하지 말고 yaml만 출력
kubectl run nginx --image=nginx --dry-run=client -o yaml

# 파일로 저장 후 수정해서 적용
kubectl create deployment web --image=nginx --dry-run=client -o yaml > web.yaml
# (web.yaml 편집) ...
kubectl apply -f web.yaml
```

> 단축 변수로 더 빠르게(→ [`08_cka-prep`](../08_cka-prep/)):
> ```bash
> export do="--dry-run=client -o yaml"
> kubectl create deploy web --image=nginx $do > web.yaml
> ```

## 필드를 모를 때 — `kubectl explain`

```bash
kubectl explain pod.spec.containers          # 필드 설명
kubectl explain pod.spec.containers.resources --recursive
```

## 원하는 정보만 뽑기 (출력 형식)

조회 결과에서 필요한 것만 보거나 추출하는 방법. **평소엔 위쪽(눈으로 보기), 자동화엔 아래쪽(값 추출).**

| 방법 | 명령 예 | 언제 쓰나 |
|---|---|---|
| **describe** | `kubectl describe pod x` | 상태·이벤트를 사람이 확인 (가장 흔함) |
| **-o wide** | `kubectl get pod -o wide` | 기본 표 + IP/노드 등 몇 개 더 |
| **-o yaml** | `kubectl get pod x -o yaml \| less` | 오브젝트 전체 구조 훑기 |
| **custom-columns** | `kubectl get pod -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image'` | 특정 필드만 표로 (CKA에 유용) |
| **jsonpath** | `kubectl get pod x -o jsonpath='{.status.podIP}'; echo` | 값 하나를 스크립트/변수로 추출 |
| **jq** | `kubectl get pod x -o json \| jq '.metadata.ownerReferences'` | 복잡한 추출·가공 (jsonpath보다 직관적) |

- 평소 조회는 **describe / -o yaml**, 가공은 **jq**. **깊은 jsonpath를 손으로 짤 일은 드물다**(학습용으로 한 번 보는 정도).
- 자동화 예: `IP=$(kubectl get pod x -o jsonpath='{.status.podIP}')`
- 자주 쓰는 jsonpath 패턴 몇 개만 익혀두면 충분:
  ```bash
  kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'   # 모든 이미지
  kubectl get nodes -o jsonpath='{.items[*].metadata.name}'             # 노드 이름들
  ```
- **CKA**: jsonpath/custom-columns 문제가 가끔 나오지만 깊고 복잡한 건 없다. 기본 패턴만 익히면 된다.

## 매니페스트(YAML) 구조

모든 오브젝트는 **4개 최상위 필드**를 가진다.

```yaml
apiVersion: apps/v1        # 어떤 API 그룹/버전 (Pod=v1, Deployment=apps/v1)
kind: Deployment           # 오브젝트 종류
metadata:                  # 이름, 네임스페이스, 라벨, 어노테이션
  name: web
  labels:
    app: web
spec:                      # 원하는 상태(desired state) — kind마다 다름
  replicas: 3
  ...
```
- `apiVersion`/`kind`를 모르면 `kubectl explain <kind>` 또는 `kubectl api-resources`로 확인.
- 적용 후 실제 상태는 `status` 필드에 채워진다(읽기 전용, 사용자가 안 씀).

## 시험·실무 팁

- `alias k=kubectl` + 자동완성 설정으로 타이핑 절약(→ [`08_cka-prep`](../08_cka-prep/)).
- 컨텍스트/네임스페이스 전환: `kubectl config use-context <c>`, `kubectl config set-context --current --namespace=<ns>`.
- `-A`(전체 네임스페이스), `-n <ns>`(특정 네임스페이스)를 습관화.
- **출력 가독성**: `-o jsonpath`/`-o go-template` 결과는 **끝에 줄바꿈이 없다.** 뒤에 `; echo`를 붙이면 결과가 제 줄에 떨어져 프롬프트와 안 붙는다.
  ```bash
  kubectl get pod nginx -o jsonpath='{.status.phase}'; echo
  ```
  (또는 jsonpath 안에 `{"\n"}`을 직접 넣어도 된다: `'...phase}{"\n"}'`)

## 참고

- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Imperative vs Declarative](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/)
- [KodeKloud 강의 노트 — Core Concepts](https://notes.kodekloud.com/docs/Certified-Kubernetes-Administrator-CKA/Core-Concepts/Imperative-vs-Declarative)
