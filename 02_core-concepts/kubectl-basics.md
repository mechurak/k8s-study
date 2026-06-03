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

## 참고

- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Imperative vs Declarative](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/)
- [KodeKloud 강의 노트 — Core Concepts](https://notes.kodekloud.com/docs/Certified-Kubernetes-Administrator-CKA/Core-Concepts/Imperative-vs-Declarative)
