# Sealed Secrets — 실습 (kind에서 손으로 해보기)

> 이 문서는 **직접 따라 하며 실행**하는 실습 가이드다. 개념은 [sealed-secrets.md](./sealed-secrets.md), 큰 그림은 [secrets-management.md](./secrets-management.md) 참고.
> 각 단계의 🔎는 **관찰 포인트**, 🧪는 **스스로 해볼 과제**다.
>
> **흐름**: 컨트롤러·CLI 설치 → ① seal→배포→자동 복호화 기본 → ② 암호문 vs base64 직접 비교 → ③ 스코프 함정(strict/namespace-wide/cluster-wide) → ④ 마스터키 백업·DR 복구 시뮬 → ⑤ 오프라인 sealing·복구(controller 없이).

## 사전 준비

kind 클러스터가 떠 있어야 한다. 환경 시작은 [`01_lab-environment`](../01_lab-environment/) 참고.

```bash
kubectl get nodes      # 노드가 Ready인지 확인
```

> 💡 이 실습은 **git에 push할 필요 없이** 로컬에서 전부 돌아간다(ArgoCD 실습과 달리). `kubeseal`로 만든 암호문을 `kubectl apply`로 직접 넣어 컨트롤러가 복호화하는 걸 본다. GitOps에선 이 "apply" 자리를 ArgoCD sync가 대신할 뿐, 원리는 같다.

---

## 실습 0 — 컨트롤러 + `kubeseal` CLI 설치

### (가) 컨트롤러 설치 (클러스터 안)

raw 매니페스트로 설치하면 컨트롤러가 **`kube-system`에 `sealed-secrets-controller`** 라는 이름으로 떠서, [sealed-secrets.md](./sealed-secrets.md)의 점검 명령들과 그대로 맞는다.

```bash
# 최신 버전 태그는 릴리스 페이지에서 확인: https://github.com/bitnami-labs/sealed-secrets/releases
VERSION=v0.27.1     # ← 최신으로 바꿔도 됨
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/${VERSION}/controller.yaml

# 컨트롤러가 뜰 때까지 대기
kubectl -n kube-system rollout status deploy/sealed-secrets-controller
kubectl -n kube-system get pods -l name=sealed-secrets-controller
```
🔎 `sealed-secrets-controller` 파드가 `Running`. 이 컨트롤러가 **키쌍을 생성**하고 `SealedSecret`을 감시하다 복호화한다.

```bash
# 컨트롤러가 자동 생성한 마스터키(개인키) — 지극히 민감
kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key
```
🔎 `Opaque` 타입 Secret 하나가 보인다. **이게 sealing key(마스터키)** 다. 처음엔 1개(키 갱신 30일마다 늘어난다).

> 🛠 **helm으로 설치하는 대안**(실무에서 흔함):
> ```bash
> helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
> helm install sealed-secrets -n kube-system sealed-secrets/sealed-secrets
> ```
> 단 helm 릴리스명이 `sealed-secrets`라 컨트롤러 이름이 달라진다 → 이후 `kubeseal`에 `--controller-name=sealed-secrets --controller-namespace=kube-system`을 붙여야 한다. **이 실습은 raw 매니페스트 기준**으로 진행한다(기본 이름이라 플래그 불필요).

### (나) `kubeseal` CLI 설치 (로컬)

```bash
# macOS
brew install kubeseal

# Linux (릴리스 바이너리) — 위 VERSION과 맞춘다
curl -sLo kubeseal.tar.gz \
  "https://github.com/bitnami-labs/sealed-secrets/releases/download/${VERSION}/kubeseal-${VERSION#v}-linux-amd64.tar.gz"
tar -xzf kubeseal.tar.gz kubeseal && sudo install -m 0755 kubeseal /usr/local/bin/kubeseal

kubeseal --version
```
🔎 CLI 버전이 컨트롤러 버전과 비슷하면 좋다(너무 차이 나면 호환 경고가 날 수 있음).

---

## 실습 1 — seal → 배포 → 컨트롤러가 자동 복호화

핵심 워크플로다. **평문 Secret은 클러스터에 절대 apply하지 않고**(`--dry-run=client`), 내 손에서 바로 `kubeseal`로 암호화한다.

```bash
# 작업용 NS
kubectl create namespace ss-demo

# 1) 평문 Secret을 "매니페스트로만" 만들고 → 바로 kubeseal로 암호화
kubectl create secret generic db-cred \
  --from-literal=username='appuser' \
  --from-literal=password='S3cr3t-Pa55' \
  --namespace ss-demo \
  --dry-run=client -o yaml \
  | kubeseal -o yaml > sealed-db-cred.yaml

# 2) 결과물은 암호문 → git에 올려도 안전. 클러스터에 적용
kubectl apply -f sealed-db-cred.yaml

# 3) 컨트롤러가 복호화해 동명의 Secret을 만든다
kubectl -n ss-demo get sealedsecret,secret
```
🔎 `SealedSecret/db-cred`와 그게 만들어낸 `Secret/db-cred`가 같이 보인다. **내가 만든 건 SealedSecret뿐**인데 Secret이 따라 생겼다 — 소유 관계(SealedSecret이 소스).

```bash
kubectl -n ss-demo describe sealedsecret db-cred   # Events에 "Sealed/Unsealed" 성공 로그
```
🔎 Events에 복호화 성공이 찍힌다. 실패하면 여기와 컨트롤러 로그에 이유가 나온다.

🧪 **과제**: SealedSecret을 지우면 어떻게 될까? `kubectl -n ss-demo delete sealedsecret db-cred` 후 `get secret db-cred` — Secret도 따라 사라진다(소유 관계). 다시 `apply -f sealed-db-cred.yaml`로 되살려 둔다.

---

## 실습 2 — 기존 SealedSecret 값만 갱신 (`--merge-into` / `--raw`)

SealedSecret의 암호문은 **손으로 못 고친다**(텍스트를 바꾸면 복호화가 깨진다). 로컬엔 개인키가 없어 **기존 값을 풀어볼 수도 없다.** 그래서 "다시 seal"하는데, 파일을 통째로 새로 만들지 않고 **바꿀 키만 갈아끼우는** 방법이 있다.

### (가) `--merge-into` — 그 키만 교체 (제일 흔함)

실습 1의 `sealed-db-cred.yaml`은 그대로 두고, `password`만 새로 암호화해 **병합**한다. `username` 등 다른 키는 건드리지 않는다(풀어볼 필요도 없음).

```bash
kubectl create secret generic db-cred \
  --from-literal=password='new-S3cr3t' \
  --namespace ss-demo \
  --dry-run=client -o yaml \
  | kubeseal --merge-into sealed-db-cred.yaml
```
🔎 `sealed-db-cred.yaml`의 `encryptedData.password`만 새 암호문으로 바뀌고 나머지는 유지된다. **이름·네임스페이스를 기존과 똑같이** 줘야 한다(strict 스코프라 안 맞으면 복호화 거부) — `--namespace`를 빠뜨리지 말 것.

```bash
# 적용(또는 ArgoCD sync) → 컨트롤러가 새 값으로 Secret 갱신
kubectl apply -f sealed-db-cred.yaml
kubectl -n ss-demo get secret db-cred -o jsonpath='{.data.password}' | base64 -d; echo   # new-S3cr3t
```
🔎 `new-S3cr3t`로 갱신됐다. 같은 방식으로 **없는 키를 주면 추가**된다(있으면 갱신, 없으면 추가).

### (나) `--raw` — 값 하나만 암호화해 손으로 붙이기

파일의 한 줄만 정밀하게 바꿀 때. 암호문 문자열만 출력되니 `encryptedData`의 해당 키 값에 붙인다.

```bash
echo -n 'another-secret' | kubeseal --raw --name db-cred --namespace ss-demo
# → AgB2k9...  (이 문자열을 sealed-db-cred.yaml의 encryptedData.<키> 값으로 교체)
```
> ⚠️ `--raw`는 **스코프를 직접 맞춰야** 한다. strict가 아니면 `--scope namespace-wide`(이땐 `--name` 생략) / `--scope cluster-wide`. 잘못 주면 `no key could decrypt`.

> 💡 어느 방법이든 **git에 커밋해야 반영**된다(파일이 곧 소스). in-cluster 객체를 직접 고치는 게 아니라 **파일을 고쳐 push → apply/sync** 흐름임을 기억(이게 GitOps). 이 "사용자가 직접 seal하지 않고 CI가 대신" 하는 방식은 실습 7에서 다룬다.

🧪 **과제**: `--merge-into`로 `apikey=xyz`를 db-cred에 **추가**해 보고(`get secret db-cred -o yaml`로 키가 3개가 됐는지 확인), 다시 `--merge-into`로 그 `apikey` 값을 바꿔 본다.

---

## 실습 3 — "암호문 vs base64" 눈으로 비교

왜 SealedSecret은 git에 올려도 되고 기본 Secret은 안 되는지 직접 본다.

```bash
# (A) 기본 Secret — base64일 뿐, 누구나 디코드된다
kubectl -n ss-demo get secret db-cred -o jsonpath='{.data.password}' | base64 -d; echo
```
🔎 평문(실습 2에서 갱신했다면 `new-S3cr3t`)이 **그대로 복원**된다. base64는 암호화가 아니다 — 이게 평문 Secret을 git에 못 두는 이유.

```bash
# (B) SealedSecret — encryptedData는 비대칭 암호문
grep -A3 encryptedData sealed-db-cred.yaml
```
🔎 `password:` 자리에 알아볼 수 없는 긴 암호문이 들어 있다. 공개키로 잠갔으므로 **컨트롤러의 개인키 없이는 못 푼다**. 그래서 git에 올려도 안전.

🧪 **과제**: Pod가 이 Secret을 실제로 쓰는지 확인. 아래를 apply하고 env가 주입되는지 본다.
```bash
kubectl -n ss-demo run user-check --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"c","image":"busybox:1.36","command":["sh","-c","echo USER=$DB_USER; sleep 3600"],"env":[{"name":"DB_USER","valueFrom":{"secretKeyRef":{"name":"db-cred","key":"username"}}}]}]}}'
kubectl -n ss-demo logs user-check     # USER=appuser
kubectl -n ss-demo delete pod user-check
```

---

## 실습 4 — 스코프 함정 (strict / namespace-wide / cluster-wide)

가장 잘 데는 곳이다. SealedSecret은 기본 **strict** — 이름+네임스페이스에 고정돼, 나중에 위치를 바꾸면 복호화가 거부된다.

### (가) strict에서 이름을 바꾸면 깨진다

```bash
# 실습 1의 sealed-db-cred.yaml(strict)에서 이름만 바꿔본다
sed 's/name: db-cred/name: db-cred-renamed/' sealed-db-cred.yaml \
  | kubectl apply -f - --namespace ss-demo
kubectl -n ss-demo describe sealedsecret db-cred-renamed | grep -i -A2 event
```
🔎 `no key could decrypt secret` — 암호문은 멀쩡하지만 **이름이 sealing 당시와 달라** 컨트롤러가 거부한다. 탈취한 암호문을 다른 이름/NS로 옮겨 푸는 걸 막는 안전장치.

```bash
kubectl -n ss-demo delete sealedsecret db-cred-renamed   # 정리
```

### (나) namespace-wide / cluster-wide로 결합도 풀기

```bash
# namespace-wide: 같은 NS 안에서는 이름을 바꿔도 OK
kubectl create secret generic flexible --from-literal=k=v \
  --namespace ss-demo --dry-run=client -o yaml \
  | kubeseal --scope namespace-wide -o yaml > sealed-flexible.yaml
kubectl apply -f sealed-flexible.yaml
sed 's/name: flexible/name: flexible2/' sealed-flexible.yaml | kubectl apply -f - --namespace ss-demo
kubectl -n ss-demo get secret flexible flexible2     # 둘 다 복호화됨
```
🔎 이번엔 이름을 바꿔도 `flexible2`가 만들어진다(annotation `sealedsecrets.bitnami.com/namespace-wide: "true"`가 붙어 있다).

```bash
# cluster-wide: 어느 NS·어느 이름이든 OK (여러 NS 공유 시크릿)
kubectl create namespace ss-demo2
kubectl create secret generic shared --from-literal=apikey=xxx \
  --dry-run=client -o yaml \
  | kubeseal --scope cluster-wide -o yaml > sealed-shared.yaml
kubectl apply -f sealed-shared.yaml --namespace ss-demo2     # 다른 NS에 넣어도 풀린다
kubectl -n ss-demo2 get secret shared
```
🔎 cluster-wide는 어디서나 풀리지만 **그만큼 덜 안전**(암호문 유출 시 아무 데서나 복호화 가능). 기본 strict를 쓰고, 여러 NS 공유는 **strict + [Reflector](./reflector.md) 복제**가 이 레포의 권장 방식.

🧪 **과제**: 현재 클러스터의 공개 인증서를 받아, 스코프가 annotation으로 어떻게 표시되는지 본다.
```bash
grep -i scope sealed-flexible.yaml sealed-shared.yaml      # namespace-wide / cluster-wide annotation
```

---

## 실습 5 — 마스터키 백업 & DR 복구 시뮬 (가장 중요)

**마스터키를 잃으면 git의 모든 SealedSecret을 영영 못 푼다.** 백업 → 키 삭제로 재난 흉내 → 복원의 한 사이클을 돌려 본다.

### (가) 백업 (라벨로 묶인 키 "전부")

```bash
# 활성 sealing key 전부 백업 (지극히 민감 — 안전한 곳에만, git 금지)
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > master.key
ls -l master.key
```
🔎 키 갱신(30일)으로 키가 여러 개가 되면 **하나만 백업하면 옛 키로 만든 게 복구 안 된다** → 항상 라벨로 전부 뜬다.

### (나) 재난 시뮬 — 키를 날리고 컨트롤러 재기동

> ⚠️ 학습용 데모 NS에서만. 실제 운영 클러스터에서 하지 말 것.

```bash
# 마스터키 삭제 → 컨트롤러가 새 키쌍을 만들어 버린다(옛 암호문과 불일치)
kubectl delete secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key
kubectl -n kube-system rollout restart deploy/sealed-secrets-controller
kubectl -n kube-system rollout status deploy/sealed-secrets-controller

# 옛 SealedSecret을 다시 적용해 보면…
kubectl delete -f sealed-db-cred.yaml --namespace ss-demo 2>/dev/null
kubectl apply -f sealed-db-cred.yaml
kubectl -n ss-demo describe sealedsecret db-cred | grep -i -A2 event
```
🔎 `no key could decrypt secret` — **새 키쌍**으론 옛 공개키로 만든 암호문을 못 푼다. "클러스터 재구축 후 전부 복호화 실패"가 정확히 이 상황.

### (다) 복원 — 백업 키를 되돌리고 재기동

```bash
kubectl apply -f master.key                                   # 옛 마스터키 복원
kubectl -n kube-system rollout restart deploy/sealed-secrets-controller
kubectl -n kube-system rollout status deploy/sealed-secrets-controller

# 잠시 후 옛 SealedSecret이 다시 복호화된다
kubectl -n ss-demo get secret db-cred
```
🔎 키를 되돌리니 옛 암호문이 다시 풀린다. **이 키 백업이 곧 안전망.** 실무 스택에선 이 `master.key`를 다시 [SOPS/age](./sops-age.md)로 암호화해 둔다("키를 지키는 키"). 전체 순서 → [secrets-dr.md](./secrets-dr.md).

> 🔐 끝나면 로컬의 `master.key`는 안전하게 지운다: `shred -u master.key`(Linux) / `rm -P master.key`(mac).

---

## 실습 6 — 오프라인 sealing & 복구 (컨트롤러 없이)

CI 등 컨트롤러에 붙을 수 없는 환경, 그리고 클러스터 자체가 없을 때의 복호화를 본다.

```bash
# (A) 오프라인 sealing — 공개 인증서를 미리 받아 --cert로 sealing (클러스터 접근 불필요)
kubeseal --fetch-cert > pub-cert.pem
kubectl create secret generic offline-demo --from-literal=token=abc123 \
  --namespace ss-demo --dry-run=client -o yaml \
  | kubeseal --cert pub-cert.pem -o yaml > sealed-offline.yaml
kubectl apply -f sealed-offline.yaml
kubectl -n ss-demo get secret offline-demo
```
🔎 `--fetch-cert`로 받은 **공개키만으로** sealing이 된다. 공개키는 유출돼도 안전(그걸론 못 푼다) → CI 파이프라인에서 안전하게 seal 가능.

```bash
# (B) 오프라인 복호화 — 백업한 개인키로 클러스터/컨트롤러 없이 푼다
kubeseal --recovery-unseal --recovery-private-key master.key < sealed-offline.yaml
# 키 파일 여러 개면 콤마로: --recovery-private-key key1.key,key2.key
```
🔎 컨트롤러가 없어도 **백업 개인키만 있으면** 평문 Secret YAML이 복원된다. 실습 5에서 master.key를 지웠다면 다시 백업을 떠서 시도.

🧪 **과제**: `kubeseal --re-encrypt`로 기존 SealedSecret을 최신 키로 재암호화해 본다(키 회전 후 옛 키를 폐기하려는 상황).
```bash
kubeseal --re-encrypt < sealed-db-cred.yaml > tmp.yaml && mv tmp.yaml sealed-db-cred.yaml
# in-cluster 객체는 안 바뀐다 → 결과를 git에 커밋해야 반영(GitOps)
```

---

## 실습 7 — CI가 대신 seal하기 (사용자는 kubeseal·kubectl 불필요, GitHub Actions)

여기까진 **내가** `kubeseal`을 돌렸다. 하지만 팀에 CLI를 못/안 쓰는 사용자가 있다면, **seal을 CI로 옮긴다.** 사용자는 GitOps(값 제공 + 트리거)만 하면 된다.

> 핵심 원리(실습 6에서 봤다): 공개 인증서는 **유출돼도 안전**하니 **저장소에 같이 둔다** → CI가 `--cert`로 **오프라인 sealing**(클러스터 접근 불필요). 사용자는 시크릿 **값**만 안전한 채널로 주면 된다.

### 전체 흐름

```mermaid
flowchart LR
    u["사용자<br/>(CLI 없음)"] -->|"① 값을 Actions Secret에 등록<br/>+ Run workflow 클릭"| gh["GitHub Actions<br/>(self-hosted runner)"]
    gh -->|"② pub-cert.pem으로 kubeseal --merge-into"| ss["SealedSecret 갱신"]
    ss -->|"③ 커밋/PR"| git["Git 저장소"]
    git -->|"④ sync"| argo["ArgoCD"]
    argo -->|"⑤ controller가 복호화"| sec["Secret (런타임)"]
```

### (가) 준비 — 공개 인증서를 저장소에 둔다 (한 번)

```bash
# 관리자가 현재 클러스터 공개키를 받아 커밋 (유출돼도 안전한 공개키)
kubeseal --fetch-cert > pub-cert.pem
git add pub-cert.pem && git commit -m "add sealing public cert"
```
🔎 이 인증서로 **누구나/무엇이나 seal**할 수 있지만 복호화는 못 한다. CI가 이걸 `--cert`로 쓴다.

### (나) 워크플로 — 값을 받아 seal하고 커밋

저장소에 `.github/workflows/seal-secret.yml`로 둔다. 사용자는 **Actions 탭 → Run workflow**로 입력만 채운다.

```yaml
name: Seal secret
on:
  workflow_dispatch:           # 사용자가 UI 버튼으로 실행
    inputs:
      name:      { description: "Secret 이름 (예: db-cred)", required: true }
      namespace: { description: "네임스페이스 (예: myapp)", required: true }
      key:       { description: "갱신할 키 (예: password)", required: true }
      path:      { description: "SealedSecret 파일 경로", required: true }

jobs:
  seal:
    runs-on: self-hosted       # 온프렘이면 보통 self-hosted runner
    steps:
      - uses: actions/checkout@v4

      - name: Install kubeseal
        run: |
          VERSION=v0.27.1
          curl -sLo ks.tgz "https://github.com/bitnami-labs/sealed-secrets/releases/download/${VERSION}/kubeseal-${VERSION#v}-linux-amd64.tar.gz"
          tar -xzf ks.tgz kubeseal && sudo install -m0755 kubeseal /usr/local/bin/kubeseal

      - name: Seal & merge into file
        env:
          SECRET_VALUE: ${{ secrets.NEW_SECRET_VALUE }}   # 웹 UI(Settings→Secrets→Actions)에서 등록 — 로그에 마스킹됨
        run: |
          kubectl create secret generic "${{ inputs.name }}" \
            --from-literal="${{ inputs.key }}=${SECRET_VALUE}" \
            --namespace "${{ inputs.namespace }}" \
            --dry-run=client -o yaml \
            | kubeseal --cert pub-cert.pem --merge-into "${{ inputs.path }}"

      - name: Commit (PR 권장)
        run: |
          git config user.name  "seal-bot"
          git config user.email "seal-bot@users.noreply.github.com"
          git add "${{ inputs.path }}"
          git commit -m "chore: rotate ${{ inputs.namespace }}/${{ inputs.name }}.${{ inputs.key }}"
          git push
```

🔎 동작: 사용자가 새 값을 **Actions Secret(`NEW_SECRET_VALUE`)** 에 넣고 워크플로를 실행 → CI가 `pub-cert.pem`으로 seal해 **SealedSecret만 갱신·커밋** → ArgoCD가 sync → controller가 복호화. **사용자는 kubeseal·kubectl을 만지지 않는다.** (`kubectl create --dry-run=client`은 YAML 생성기일 뿐 클러스터 접근이 없다 — runner에 바이너리만 있으면 됨.)

### 실무 포인트 (꼭)

- **보호 브랜치면 직접 push 대신 PR**로. 시크릿 변경은 리뷰 대상이다 → `peter-evans/create-pull-request` 액션으로 브랜치+PR 생성하게 바꾼다.
- **값은 로그에 절대 노출 금지.** Actions Secret으로 주입하면 자동 마스킹된다. `echo "$SECRET_VALUE"` 같은 건 쓰지 말 것.
- **인증서 갱신**: sealing key는 30일마다 갱신된다(실습 5). 저장소의 `pub-cert.pem`도 **주기적으로 다시 `--fetch-cert`** 해 최신으로 유지(옛 인증서로 seal해도 그 키가 활성 집합에 있는 동안은 복호화되지만, 최신이 안전).
- **여러 시크릿을 다룬다면** Actions Secret 한 칸을 돌려쓰기보다 **Environment별 시크릿**이나 외부 입력 채널(사내 Vault 등)을 쓰는 게 깔끔하다. 위 예시는 "한 값 회전"의 최소 형태다.
- **그래도 seal 단계는 사라지지 않는다** — CI라는 자동화로 옮겼을 뿐. 사용자가 값 입력조차 CLI 없이 하려면 모델 자체를 External Secrets Operator로 바꾸는 선택지가 있다(값을 콘솔/스토어에 넣고 git엔 참조만) → [secrets-management.md](./secrets-management.md), 실무는 [09_aws-eks](../09_aws-eks/).

---

## 정리 (실습 후 청소)

```bash
# 데모 NS 정리 (SealedSecret·Secret 함께 사라짐)
kubectl delete namespace ss-demo ss-demo2

# 로컬에 만든 파일 정리 — 특히 master.key는 안전 삭제
rm -f sealed-*.yaml pub-cert.pem
shred -u master.key 2>/dev/null || rm -f master.key

# 컨트롤러까지 걷어내기 (원하면)
kubectl delete -f https://github.com/bitnami-labs/sealed-secrets/releases/download/${VERSION}/controller.yaml
```
> ⚠️ 컨트롤러를 지우면 마스터키(Secret)도 사라진다. 같은 SealedSecret을 다시 쓰려면 **백업한 키를 복원**해야 함을 기억(실습 5).

---

## 한 장 요약 (체득 포인트)

| 무엇을 배웠나 | 핵심 |
|---|---|
| 기본 워크플로 | `create secret --dry-run=client \| kubeseal` → apply → 컨트롤러가 자동 복호화 |
| 값만 갱신 | `--merge-into`(그 키만 교체) / `--raw`(한 줄) → 파일 고쳐 커밋 (in-cluster 직접수정 아님) |
| CLI 없는 사용자 | 값을 Actions Secret에 등록 → **CI가 `--cert`로 대신 seal·커밋** → ArgoCD sync |
| 왜 git에 안전한가 | SealedSecret은 **비대칭 암호문**, 기본 Secret은 **base64**(디코드됨) |
| 스코프 함정 | strict(기본)는 name+NS 고정 → 위치 바꾸면 `no key could decrypt` |
| 운영 1순위 | **마스터키 백업**(라벨로 전부). 잃으면 복구 불가 |
| 컨트롤러 없이도 | `--fetch-cert`로 오프라인 sealing, `--recovery-unseal`로 오프라인 복호화 |

## 참고

- [sealed-secrets.md](./sealed-secrets.md) — 이 폴더의 개념 정리 · [secrets-management.md](./secrets-management.md) — 전체 그림
- [Sealed Secrets — README / Releases](https://github.com/bitnami-labs/sealed-secrets/releases)
- [kubeseal 사용법·스코프·복구 문서](https://github.com/bitnami-labs/sealed-secrets#usage)
