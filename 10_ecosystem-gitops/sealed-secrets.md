# Sealed Secrets — 시크릿을 git에 암호화해서 둔다

> 런타임 시크릿을 **공개키로 암호화한 `SealedSecret`** 으로 바꿔 git에 안전히 올리고, 클러스터 안 controller만 **개인키로 복호화**해 평범한 `Secret`을 만든다.
> 큰 그림·다른 도구와의 관계 → [secrets-management.md](./secrets-management.md) · 복구 런북 → [secrets-dr.md](./secrets-dr.md)
> **직접 따라 하는 실습(kind)** → [practice-sealed-secrets.md](./practice-sealed-secrets.md)

## 왜 필요한가 — `Secret`은 암호화가 아니라 base64다

GitOps는 "git이 단일 소스"가 원칙인데, k8s 기본 `Secret`은 값이 **base64로 인코딩만** 됐을 뿐 누구나 디코드할 수 있다. 그대로 git에 올리면 비밀번호를 평문으로 공개하는 셈. Sealed Secrets는 이걸 **되돌릴 수 없는 방향의 암호(비대칭)** 로 풀어, "암호문은 git에, 복호화 능력은 클러스터 안에만" 둔다.

```mermaid
flowchart LR
    plain["평문 Secret<br/>(내 노트북)"]
    plain -->|"kubeseal<br/>(공개키로 암호화)"| sealed["SealedSecret<br/>(암호문, git에 안전)"]
    sealed -->|"git push / ArgoCD sync"| git["Git 저장소"]
    git --> ctl["sealed-secrets controller<br/>(클러스터 안, 개인키 보유)"]
    ctl -->|"개인키로 복호화"| secret["Secret (런타임)"]
    secret --> pod["Pod가 마운트/env로 사용"]
```

- **공개키로 잠그고(seal), 개인키로만 연다(unseal).** 공개키는 내보내도 안전 — 그걸로는 못 푼다. 그래서 누구나 sealing할 수 있지만 복호화는 클러스터만 가능.
- **암호문은 특정 클러스터 전용**이다. A 클러스터 공개키로 만든 SealedSecret은 B 클러스터(다른 키쌍)에선 못 풀린다.

## 구성 요소 — CLI 하나 + 클러스터 안 컨트롤러 하나

| 요소 | 형태 | 역할 |
|---|---|---|
| **`kubeseal`** | 로컬 CLI | 평문 Secret을 **공개키로 암호화** → `SealedSecret` YAML 생성 |
| **controller** | 클러스터 안 Deployment (`kube-system`, `name=sealed-secrets-controller`) | 키쌍 관리 + `SealedSecret`을 감시하다 **개인키로 복호화** → `Secret` 생성·갱신 |
| **`SealedSecret`** | CRD (`bitnami.com/v1alpha1`) | git에 올리는 **암호문 매니페스트**. 복호화되면 동명의 `Secret`이 생김 |

> SealedSecret을 지우면 controller가 만든 Secret도 따라 지워진다(소유 관계). Secret을 사람이 직접 만든 게 아니라 **SealedSecret이 곧 소스**.

## 암호화 방식 — 하이브리드(대칭+비대칭)

큰 데이터를 RSA로 직접 암호화하면 느리고 크기 제한이 있어, **하이브리드**를 쓴다:

1. 임의의 **세션키**로 시크릿 데이터를 **AES-256-GCM**(대칭) 암호화 — 빠르고 크기 무관.
2. 그 세션키를 controller의 **공개키(RSA-4096)** 로 **RSA-OAEP(SHA-256)** 암호화.
3. 둘을 묶어 `SealedSecret`에 담는다. 복호화는 controller가 개인키로 세션키를 풀고 → 세션키로 데이터를 푼다.

> 외울 필요는 없지만, "공개키로 직접 다 암호화하는 게 아니라 세션키를 거친다"는 점만 알면 동작이 이해된다.

## 기본 사용 — seal → push → 클러스터가 알아서

```bash
# 1) 평문 Secret을 (적용하지 말고) 매니페스트로만 만들고, 바로 kubeseal에 파이프
kubectl create secret generic mysecret \
  --from-literal=password='db-password-123' \
  --dry-run=client -o yaml \
  | kubeseal -o yaml > sealed-mysecret.yaml

# 2) 결과물(sealed-mysecret.yaml)은 암호문 → git에 올려도 안전
git add sealed-mysecret.yaml && git commit -m "add sealed db secret"

# 3) 클러스터에 적용(또는 ArgoCD가 sync) → controller가 복호화해 Secret 생성
kubectl apply -f sealed-mysecret.yaml
kubectl get secret mysecret           # 복호화된 런타임 Secret이 생겨 있어야
```

- 평문 Secret은 **클러스터에 apply하지 않는다**(`--dry-run=client`). 평문은 내 손에서 kubeseal로 바로 들어가고, git엔 암호문만.
- **오프라인 sealing**: controller에 접근 못 하는 환경(CI 등)에선 공개 인증서를 미리 받아 `--cert`로 쓴다.
  ```bash
  kubeseal --fetch-cert > pub-cert.pem      # 현재 클러스터 공개키 받기
  kubeseal --cert pub-cert.pem -o yaml < secret.yaml > sealed.yaml
  ```

## 기존 값만 갱신 — 손으로 못 고친다, 다시 seal한다

SealedSecret의 암호문은 **직접 편집 불가**(텍스트를 바꾸면 복호화가 깨진다)이고, 로컬엔 개인키가 없어 **기존 값을 풀어볼 수도 없다.** 그래서 바꿀 키만 **다시 seal해 갈아끼운다**:

```bash
# (A) --merge-into: 기존 파일에 그 키만 병합 (다른 키는 유지·풀어볼 필요 없음) — 제일 흔함
kubectl create secret generic db-cred --from-literal=password='new' \
  --namespace myapp --dry-run=client -o yaml | kubeseal --merge-into sealed-db-cred.yaml

# (B) --raw: 값 하나만 암호화해 encryptedData에 손으로 붙임 (스코프 직접 지정)
echo -n 'new' | kubeseal --raw --name db-cred --namespace myapp
```
- 이름·네임스페이스를 **기존과 동일**하게(strict면 안 맞으면 복호화 거부). 결과는 **git에 커밋해야 반영**(파일이 곧 소스).

## CI/파이프라인으로 seal — 사용자가 kubeseal·kubectl을 안 쓰게

seal은 **누군가 `kubeseal`을 돌려야** 하는 단계라 순수 git 편집만으론 SealedSecret을 만들 수 없다. 하지만 그 `kubeseal`을 **사용자 노트북이 아니라 CI에 두면**, 사용자는 GitOps(값 제공 + 트리거/PR)만 하면 된다.

- 공개 인증서(`pub-cert.pem`)는 **유출돼도 안전**하니 저장소에 둔다 → CI가 `--cert`로 **오프라인 sealing**(클러스터 접근 불필요).
- 사용자는 시크릿 **값**만 안전한 채널(예: CI 시크릿)로 제공 → CI가 seal → SealedSecret 커밋/PR → ArgoCD sync. (반영(apply)은 ArgoCD가 하므로 **이미 순수 GitOps**다.)
- 키 갱신(30일)으로 인증서가 바뀌니 저장소의 `pub-cert.pem`도 주기적으로 갱신한다.
- CLI는 물론 **값 입력조차** 안 시키려면 모델을 **External Secrets Operator**로 바꾼다(값은 콘솔/스토어에, git엔 참조만) → [secrets-management.md](./secrets-management.md).

> 동작하는 **GitHub Actions 예시**(GitHub Enterprise 포함)는 [practice-sealed-secrets.md 실습 7](./practice-sealed-secrets.md)에 있다.

## 스코프 — 암호문을 어디까지 재사용할 수 있나 (함정 주의)

SealedSecret은 기본적으로 **이름+네임스페이스에 고정**된다. 이름이나 NS를 바꾸면 복호화가 거부된다(탈취한 암호문을 다른 데로 옮겨 푸는 걸 막는 안전장치). 스코프로 이 결합도를 조절한다:

| 스코프 | 무엇에 고정 | 바꿔도 되나 | annotation | 언제 |
|---|---|---|---|---|
| **strict** (기본) | name + namespace | 둘 다 못 바꿈 | (없음) | 대부분. 가장 안전 |
| **namespace-wide** | namespace만 | 같은 NS면 **이름 변경 OK** | `sealedsecrets.bitnami.com/namespace-wide: "true"` | 한 NS 안에서 이름 유연하게 |
| **cluster-wide** | (없음) | **어느 NS·어느 이름이든 OK** | `sealedsecrets.bitnami.com/cluster-wide: "true"` | 여러 NS가 공유하는 시크릿 |

```bash
# 예: cluster-wide로 sealing
kubectl create secret generic shared --from-literal=apikey=xxx --dry-run=client -o yaml \
  | kubeseal --scope cluster-wide -o yaml > sealed-shared.yaml
```

> ⚠️ **제일 잘 데는 곳**: strict로 sealing해 놓고 나중에 SealedSecret의 `metadata.name`/`namespace`를 바꾸면 **"no key could decrypt"** 로 실패한다. 이름/NS를 바꿀 가능성이 있으면 스코프를 맞춰 sealing하거나, 바뀐 위치로 **다시 seal**해야 한다.
> 여러 NS에서 같은 시크릿을 써야 한다면 cluster-wide 대신 **strict + [Reflector](./reflector.md)로 복제**하는 선택지도 있다(이 레포 스택의 방식).

## 마스터키(sealing key) — 이게 전부다

복호화에 쓰는 **개인키 = sealing key**. 클러스터를 잃어도 git의 SealedSecret을 다시 풀려면 이 키가 있어야 한다. **키를 잃으면 git의 모든 SealedSecret은 영영 못 푼다.**

- `kube-system` 네임스페이스에 Secret으로 저장되며 라벨이 붙는다: `sealedsecrets.bitnami.com/sealed-secrets-key`.
- **자동 갱신**: 기본 **30일마다** 새 sealing key를 만들어 *활성 키 집합에 추가*한다(옛 키는 복호화용으로 계속 보관). 새 시크릿은 **가장 최근 키**로 sealing된다. → `--key-renew-period`로 조정(예: `0`이면 자동 갱신 끔).
- 갱신으로 키가 늘어나므로, **백업은 "라벨로 묶인 키 전부"** 를 떠야 한다(하나만 뜨면 옛 키로 만든 게 복구 안 됨):

```bash
# 활성 sealing key 전부 백업 (지극히 민감 — 안전한 곳에만)
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > main.key
# 키 갱신이 일어날 때마다 이 백업을 다시 떠야 한다
```

> 🔐 **이 `main.key`를 어떻게 보관하나가 보안의 핵심.** 이 레포 스택에선 이 마스터키 백업을 다시 **[SOPS/age](./sops-age.md)로 암호화**해 둔다("키를 지키는 키"). 복구 순서는 [secrets-dr.md](./secrets-dr.md) 참고.

## 키 회전 · 재암호화

- sealing key가 갱신돼도 **옛 SealedSecret은 그대로 동작**한다(옛 개인키를 controller가 계속 보관하므로). 당장 다시 seal할 필요 없음.
- 더 강하게 회전하고 싶을 때(옛 키를 폐기하려면) 기존 SealedSecret을 **최신 키로 재암호화**:
  ```bash
  kubeseal --re-encrypt < my-sealed.yaml > tmp.yaml && mv tmp.yaml my-sealed.yaml
  # in-cluster 객체는 안 바뀜 → 결과를 git에 커밋해야 반영
  ```

## 복구 — controller 없이도 풀 수 있다

백업한 개인키만 있으면 클러스터/컨트롤러가 없어도 오프라인 복호화가 된다:

```bash
kubeseal --recovery-unseal --recovery-private-key main.key < my-sealed.yaml
# 여러 키 파일은 콤마로: --recovery-private-key key1.key,key2.key
```

전체 DR 절차(age→SOPS로 마스터키 복원 → controller 주입 → 재기동)는 [secrets-dr.md](./secrets-dr.md).

## 점검 / 트러블슈팅

```bash
kubectl -n kube-system get pods -l name=sealed-secrets-controller     # 컨트롤러 살아있나
kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key  # 마스터키(개수=갱신 횟수)
kubectl get sealedsecret -A                       # git→클러스터로 동기화됐나
kubectl describe sealedsecret <name>              # Events에 복호화 성공/실패
kubectl -n kube-system logs deploy/sealed-secrets-controller   # 복호화 에러 원인
```

| 증상 | 원인 | 해결 |
|---|---|---|
| `no key could decrypt secret` | strict인데 name/namespace를 바꿈, 또는 다른 클러스터 키로 sealing | 해당 위치로 다시 seal, 또는 스코프 조정 / `--fetch-cert`로 현재 키 확인 |
| SealedSecret은 있는데 Secret이 안 생김 | controller 다운 / CRD 미설치 | 컨트롤러 파드·로그, CRD 설치 여부 |
| 클러스터 재구축 후 전부 복호화 실패 | **마스터키 미복원** | 백업 키 복원(`kubectl apply -f main.key`) 후 컨트롤러 재기동 |
| CI에서 sealing 실패(클러스터 접근 불가) | controller에 못 붙음 | `kubeseal --fetch-cert`로 공개 인증서 받아 `--cert`로 오프라인 sealing |

## 시험·실무 팁

- **CKA 범위 아님**(부가 도구). 시험의 Secret은 기본 `Secret`(base64) 개념 → [03_workloads-scheduling](../03_workloads-scheduling/).
- **운영 1순위는 마스터키 백업.** Sealed Secrets의 안전성은 "그 개인키를 어디에 어떻게 백업했나"로 결정된다. 키 갱신(30일)마다 백업 갱신을 잊지 말 것.
- **평문 Secret을 절대 git/클러스터에 남기지 말 것** — `--dry-run=client`로 만들어 바로 kubeseal에 파이프.
- **다른 선택지와의 비교**: SOPS/age도 git 암호화가 되지만 "사람이 복호화"에 가깝고, Sealed Secrets는 "클러스터가 자동 복호화"라 GitOps 런타임에 더 맞는다. EKS 실무에선 **External Secrets Operator + AWS Secrets Manager**로 가는 경우가 많다(클라우드 KMS 위임) → [09_aws-eks](../09_aws-eks/).

## 참고

- [Sealed Secrets (bitnami-labs) — README](https://github.com/bitnami-labs/sealed-secrets)
- [공식 문서 — Scopes / Key renewal / Recovery](https://github.com/bitnami-labs/sealed-secrets/blob/main/docs/) · [FAQ](https://github.com/bitnami-labs/sealed-secrets/blob/main/site/content/docs/latest/reference/faq.md)
- 같은 폴더: [secrets-management.md](./secrets-management.md)(전체 그림) · [sops-age.md](./sops-age.md)(키 보호) · [reflector.md](./reflector.md)(NS 복제) · [secrets-dr.md](./secrets-dr.md)(복구)
