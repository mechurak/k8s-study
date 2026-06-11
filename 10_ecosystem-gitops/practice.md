# 10. GitOps — 실습 (ArgoCD로 배포하기)

> 이 문서는 **직접 따라 하며 실행**하는 실습 가이드다. 명령을 본인이 치고 결과를 관찰한다. 개념은 [argocd.md](./argocd.md) 참고.
> 각 단계의 🔎는 **관찰 포인트**, 🧪는 **스스로 해볼 과제**다.
>
> **흐름**: ArgoCD 설치 → ① 공식 예제로 첫 배포 체험 → ② 내 저장소로 nginx 배포 → ③ PostgreSQL(StatefulSet) 배포 → ④ GitOps 워크플로(push·self-heal·prune) 실험.

## 사전 준비

클러스터가 떠 있어야 한다. 실습 환경(colima·kind 등) 시작은 [`01_lab-environment`](../01_lab-environment/) 참고.

```bash
kubectl get nodes     # 노드가 Ready인지 확인
```

> 💡 **이 실습의 핵심 전제**: ArgoCD는 **로컬 디스크가 아니라 GitHub(원격 저장소)에서** 매니페스트를 읽는다. 그래서 이 폴더의 매니페스트는 **git에 push돼 있어야** ArgoCD가 볼 수 있다(→ 실습 ②에서 다룬다).

---

## 실습 0 — ArgoCD 설치하고 접속하기

### (가) 설치

```bash
# 1) argocd 네임스페이스 + 공식 설치 매니페스트 적용
kubectl create namespace argocd
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2) 핵심 컴포넌트가 다 뜰 때까지 대기 (수십 초~수 분)
kubectl -n argocd wait --for=condition=available --timeout=300s deployment --all
kubectl -n argocd get pods
```
> ⚠️ **`--server-side`를 꼭 붙인다.** 일반 `kubectl apply`(클라이언트 사이드)는 적용 내용을 `last-applied-configuration` annotation에 통째로 저장하는데, ArgoCD의 `ApplicationSet` CRD는 스키마가 커서 이 annotation이 **256KB 한도를 초과**해 다음 에러로 거부된다:
> `CustomResourceDefinition "applicationsets.argoproj.io" is invalid: metadata.annotations: Too long`
> server-side apply는 서버가 적용을 처리해 이 문제를 우회한다. 이미 일반 apply로 설치하다 위 에러를 만났다면, 빠진 CRD를 채우려 다시 실행할 때는 `--force-conflicts`를 함께 준다:
> `kubectl apply --server-side --force-conflicts -n argocd -f .../install.yaml`
🔎 `argocd-server`, `argocd-repo-server`, `argocd-application-controller`(StatefulSet), `argocd-redis`, `argocd-dex-server`가 보인다. 각각의 역할은 [argocd.md의 핵심 구성요소](./argocd.md#핵심-구성요소) 표와 맞춰 본다.

### (나) Web UI 접속

```bash
# 초기 admin 비밀번호 확인 (설치 시 자동 생성된 Secret)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo

# UI를 로컬로 터널 (이 터미널은 점유됨 — 새 터미널에서 이후 작업)
kubectl -n argocd port-forward svc/argocd-server 8080:443
```
브라우저로 **https://localhost:8080** → 자체 서명 인증서 경고는 무시(고급 → 계속). 사용자 `admin`, 비밀번호는 위에서 확인한 값.

> 📌 **이 한 줄 뜯어보기 (Secret 디코딩 + jsonpath — CKA 단골 스킬)**
> - **`-o jsonpath='{.data.password}'`** — 객체에서 **특정 필드만** 뽑는다. Secret의 `data` 값은 **항상 base64로 인코딩**돼 있어 그대로 보면 알아볼 수 없다.
> - **`|`** (파이프) — 앞 명령의 출력을 뒤 명령의 입력으로 넘긴다. 여기선 base64 문자열을 `base64`에게 전달.
> - **`base64 -d`** — 그 문자열을 **디코딩**(`-d`=decode)해 실제 비밀번호로 복원.
> - **`; echo`** — `;`는 "다음 명령 실행", `echo`는 **줄바꿈 한 번**. `base64 -d` 출력 끝에 개행이 없어 프롬프트가 비번에 딱 붙는 걸 막아 보기 좋게 한다.
>
> **jsonpath는 CKA 필수기**다 — 노드 IP·컨테이너 이미지·Secret 값 등 출력에서 원하는 필드만 뽑는 문제가 자주 나온다. 같은 일을 하는 대안:
> ```bash
> # go-template: base64decode까지 한 번에 (base64 -d 불필요)
> kubectl -n argocd get secret argocd-initial-admin-secret \
>   -o go-template='{{.data.password | base64decode}}'; echo
> # argocd CLI가 있으면 가장 간단 (디코딩 자동)
> argocd admin initial-password -n argocd
> ```

🔎 처음엔 Application이 하나도 없다. 이 화면이 "원하는 상태 vs 실제 상태"를 보여주는 대시보드다.

### (다) CLI 설치·로그인 (선택, 권장)

UI만으로도 되지만 CLI가 빠를 때가 많다.
```bash
# 설치 — macOS: brew install argocd / Ubuntu: 공식 릴리스 바이너리
brew install argocd      # mac

# 로그인 (port-forward가 떠 있는 상태에서)
PW=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)
argocd login localhost:8080 --username admin --password "$PW" --insecure
```
🔎 `argocd app list` 가 빈 목록을 반환하면 로그인 성공.

---

## 실습 1 — 공식 예제(guestbook)로 첫 Application 체험

내 저장소를 건드리기 전에, ArgoCD가 제공하는 **공개 예제 저장소**로 "Application이란 게 뭔지" 5분 만에 감을 잡는다.

### CLI로
```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

argocd app get guestbook      # 상태 확인 — 처음엔 OutOfSync
argocd app sync guestbook     # 수동 동기화 → 클러스터에 적용
```

> UI로 하려면: **+ NEW APP** → repo `https://github.com/argoproj/argocd-example-apps.git`, path `guestbook`, cluster `https://kubernetes.default.svc`, namespace `default` → CREATE → **SYNC**.

🔎 `argocd app get guestbook`에서 **Sync Status**가 `OutOfSync`→`Synced`, **Health**가 `Missing`→`Healthy`로 바뀌는 과정을 본다. UI에서는 리소스 트리(Application → Deployment → ReplicaSet → Pod)가 그려진다.
🔎 실제로 떴는지 확인: `kubectl get pods -l app.kubernetes.io/name=guestbook-ui` (또는 `kubectl get all -l app=guestbook-ui`).

🧪 정리: `argocd app delete guestbook` (또는 UI에서 DELETE). 🔎 Application을 지우면 **배포했던 리소스도 같이 정리**되는지 관찰.

---

## 실습 2 — 내 저장소의 매니페스트로 nginx 배포 (Pod 배포 체험)

이제 **내 Git 저장소를 소스**로 삼는다. 이 폴더에는 ArgoCD가 읽을 매니페스트가 이미 준비돼 있다:

- [`manifests/web/`](./manifests/web/) — nginx `Deployment`(replicas 3) + `Service`
- [`manifests/argocd-apps/web-app.yaml`](./manifests/argocd-apps/web-app.yaml) — 위를 가리키는 ArgoCD `Application`

### ⚠️ 먼저: 매니페스트를 GitHub에 올리기

ArgoCD는 GitHub에서 읽으므로, **이 파일들이 push돼 있어야** 한다.
```bash
git add 10_ecosystem-gitops/
git commit -m "10 ArgoCD 실습 매니페스트 추가"
git push
```
> `web-app.yaml`의 `repoURL`은 이미 이 저장소(`github.com/mechurak/k8s-study.git`), `targetRevision: main`으로 설정돼 있다. **fork했거나 다른 저장소면 `repoURL`을 본인 것으로 고쳐야 한다.**

### Application 적용

선언형으로, Application CR 자체를 등록한다(이것이 GitOps의 진입점 — 이게 무슨 뜻인지는 아래 박스 참고).
```bash
kubectl apply -f 10_ecosystem-gitops/manifests/argocd-apps/web-app.yaml

argocd app get web        # 또는 UI에서 web 카드 클릭
```

> 🖱️ **UI로 등록해도 같다 (사내에서 흔한 방식).** `kubectl` 대신 **+ NEW APP → EDIT AS YAML → `web-app.yaml` 내용 붙여넣기 → CREATE** 로 등록해도 **클러스터에 생기는 Application 객체는 동일**하다. 통로만 UI(argocd-server 경유)일 뿐. 회사에서는 보통 개발자에게 운영 클러스터 kubectl 자격증명 대신 **SSO로 UI만 열어주고**, ArgoCD RBAC로 통제하려고 이 방식을 가이드한다. 자세한 비교와 주의점(등록은 일회성 → App of Apps로 자동화)은 [argocd.md의 "실무 메모"](./argocd.md#실무-메모--application을-등록하는-방법과-사내-흔한-패턴) 참고.

> 📌 **"Application을 등록한다"는 건** nginx를 직접 띄우는 게 아니라, "이 Git 경로(`manifests/web`)를 보고 배포하라"는 **지시서(Application)를 클러스터에 넣는 것**이다. 실제 nginx 배포는 그 지시서를 읽은 ArgoCD가 한다. 이 최초 등록 한 번이 이후 자동 배포·self-heal을 시작시키는 **진입점**이다.

🔎 이 Application은 `syncPolicy.automated`라 **SYNC를 안 눌러도** 잠시 후 자동으로 동기화된다(폴링 최대 3분, 급하면 `argocd app sync web` 또는 UI **REFRESH/SYNC**).

```bash
# 배포 결과 확인 — web 네임스페이스는 CreateNamespace=true로 자동 생성됨
kubectl -n web get deploy,rs,pod,svc
```
🔎 Pod 3개가 `Running`. **이 Pod들을 `kubectl apply`로 직접 만든 게 아니다** — Git에 있는 선언을 ArgoCD가 대신 적용했다. 이게 push형(kubectl)과 pull형(GitOps)의 차이.

```bash
# 실제로 응답하는지 (새 터미널)
# 로컬 8080은 ArgoCD UI(실습 0)가 점유 중이므로 8081로 포워딩한다 (콜론 뒤 80은 Service 포트라 그대로)
kubectl -n web port-forward svc/web 8081:80
# 다른 터미널: curl -s localhost:8081 | head  →  "Welcome to nginx!"
```

🧪 UI에서 `web` Application을 열어 리소스 트리를 펼쳐 본다. Service → Deployment → ReplicaSet → Pod 3개의 관계가 보이는가?

---

## 실습 3 — PostgreSQL 배포 (StatefulSet · PVC · Secret)

DB처럼 **상태를 가진** 워크로드를 GitOps로 배포한다. 매니페스트는 [`manifests/postgres/`](./manifests/postgres/)에 준비돼 있다:

| 파일 | 무엇 |
|---|---|
| `secret.yaml` | DB 비밀번호 (학습용 평문 — 실무 금지) |
| `service.yaml` | headless Service (안정적 DNS 이름) |
| `statefulset.yaml` | postgres:17, `volumeClaimTemplates`로 PVC 자동 생성 |

### 배포

```bash
kubectl apply -f 10_ecosystem-gitops/manifests/argocd-apps/postgres-app.yaml
argocd app get postgres
```
```bash
kubectl -n database get statefulset,pod,pvc,svc,secret
```
🔎 관찰 포인트:
- Pod 이름이 `postgres-0` — StatefulSet은 **순번 이름**을 준다(Deployment의 랜덤 해시와 대비).
- PVC `data-postgres-0`가 자동 생성됐다 — `volumeClaimTemplates`가 Pod마다 전용 볼륨을 만든다.
- `kubectl -n database get pvc` 의 `STORAGECLASS`는 `standard`(kind 기본 local-path).

🔎 준비될 때까지: `kubectl -n database wait --for=condition=ready pod/postgres-0 --timeout=120s`. `pg_isready` readinessProbe가 통과해야 Ready가 된다.

### 진짜 DB인지 — 접속해서 데이터 넣기

```bash
kubectl -n database exec -it postgres-0 -- psql -U app -d appdb
```
psql 프롬프트(`appdb=#`)에서:
```sql
CREATE TABLE notes (id serial PRIMARY KEY, body text);
INSERT INTO notes (body) VALUES ('hello gitops'), ('postgres on k8s');
SELECT * FROM notes;
\q
```

### 🔑 핵심 실험 — Pod를 죽여도 데이터가 살아남는가

StatefulSet + PVC의 존재 이유를 직접 확인한다.
```bash
kubectl -n database delete pod postgres-0     # Pod 삭제
kubectl -n database get pvc                    # PVC는 그대로 남아 있다 (삭제되지 않음)
kubectl -n database get pod -w                 # postgres-0가 다시 생성되는 걸 관찰 (Ctrl+C로 중단)
```
Pod가 다시 `Running`이 되면, 데이터가 보존됐는지 확인:
```bash
kubectl -n database exec -it postgres-0 -- psql -U app -d appdb -c 'SELECT * FROM notes;'
```
🔎 아까 넣은 2개 행이 그대로 보인다 — Pod는 새로 떴지만 **같은 PVC를 다시 마운트**했기 때문. 만약 Deployment + `emptyDir`였다면 데이터는 사라졌을 것이다. (스토리지 개념 → [`05_storage`](../05_storage/))

🧪 다른 Pod에서 접속해 보기 — 클러스터 내부 DNS로 닿는지:
```bash
kubectl -n database run psql-client --rm -it --image=postgres:17 --restart=Never -- \
  psql -h postgres.database.svc.cluster.local -U app -d appdb -c '\dt'
# 비밀번호를 물으면 secret.yaml의 값(devpassword) 입력
```
🔎 앱은 Pod IP가 아니라 **Service 이름(`postgres.database.svc.cluster.local`)**으로 DB에 붙는다 — Pod가 재생성돼 IP가 바뀌어도 이름은 그대로다.

---

## 실습 4 — GitOps 워크플로 체험 (여기가 진짜 핵심)

ArgoCD를 쓰는 이유: **Git이 바뀌면 클러스터가 따라오고, 클러스터를 손대면 Git으로 되돌린다.** 두 방향을 직접 만든다.

### (가) Git → 클러스터 (push하면 배포된다)

`manifests/web/deployment.yaml`에서 `replicas: 3` → `5`로 바꾸고:
```bash
# 편집 후
git add 10_ecosystem-gitops/manifests/web/deployment.yaml
git commit -m "web replicas 3 -> 5"
git push
```
```bash
# ArgoCD가 변화를 감지할 때까지 기다리거나, 즉시 당겨오기
argocd app get web --refresh
kubectl -n web get pods -w
```
🔎 `kubectl`로 scale한 적이 없는데 Pod가 5개로 늘어난다 — **배포 = git push**. 이게 GitOps의 핵심 경험이다. UI에서 보면 잠깐 `OutOfSync`였다가 자동 sync 후 `Synced`로 돌아온다.

### (나) 클러스터 → Git (self-heal: 손으로 바꾸면 되돌린다)

이번엔 Git을 안 건드리고 **클러스터를 직접** 바꿔 본다:
```bash
kubectl -n web scale deploy/web --replicas=1     # 의도적 드리프트
kubectl -n web get pods -w
```
🔎 잠시 1개로 줄었다가, ArgoCD `selfHeal`이 **다시 5개로 되돌린다**(Git의 desired state가 5이므로). 운영에서 "콘솔에서 몰래 바꾼" 변경이 자동 교정되는 모습 — 강력하지만 그래서 위험하다([argocd.md 주의](./argocd.md#sync--상태를-맞추는-동작)).

### (다) prune (Git에서 지우면 클러스터에서도 사라진다)

`manifests/web/service.yaml`을 지우고 push해 본다:
```bash
git rm 10_ecosystem-gitops/manifests/web/service.yaml
git commit -m "web service 제거 (prune 실험)"
git push
argocd app sync web    # 또는 자동 sync 대기
```
🔎 `kubectl -n web get svc` — `web` Service가 사라졌다(`prune: true`라서). prune이 꺼져 있었다면 Service는 `OutOfSync`로 남되 삭제되진 않았을 것.
> 실험 후 되돌리려면 `git revert` 또는 파일 복구 후 다시 push.

---

## (보너스) App of Apps — Application도 Git으로 관리

지금까지 `web-app.yaml`·`postgres-app.yaml`을 `kubectl apply`로 직접 적용했다. 이 **Application들 자체도** 하나의 부모 Application으로 묶어 GitOps로 부트스트랩할 수 있다.

```bash
argocd app create bootstrap \
  --repo https://github.com/mechurak/k8s-study.git \
  --path 10_ecosystem-gitops/manifests/argocd-apps \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --sync-policy automated

argocd app get bootstrap
```
🔎 부모(`bootstrap`) 하나를 만들었더니 자식(`web`, `postgres`)이 자동 생성된다. UI에서 부모→자식→실제 리소스로 이어지는 트리를 본다. 개념은 [argocd.md의 App of Apps](./argocd.md#app-of-apps-패턴).

---

## 정리 (실습 후 청소)

```bash
# Application 삭제 (배포한 리소스도 함께 정리됨)
argocd app delete web postgres bootstrap guestbook 2>/dev/null
# 또는 kubectl -n argocd delete application web postgres

# 네임스페이스째 정리
kubectl delete namespace web database

# ArgoCD 자체를 걷어내기 (원하면)
kubectl delete namespace argocd
```
> ⚠️ `database` 네임스페이스를 지우면 PVC도 삭제돼 **DB 데이터가 사라진다**. 데이터를 남기려면 네임스페이스는 두고 Application만 지운다.

🔎 일상적으로 클러스터 전체를 멈추는 건 [`01_lab-environment`](../01_lab-environment/)의 `colima stop`(macOS) 루틴을 쓴다 — ArgoCD·PVC 모두 보존된다.

---

## 참고

- [argocd.md](./argocd.md) — 이 폴더의 개념 정리
- [Argo CD – Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [argocd-example-apps (guestbook 등)](https://github.com/argoproj/argocd-example-apps)
- [PostgreSQL on Kubernetes 공식 이미지](https://hub.docker.com/_/postgres)
