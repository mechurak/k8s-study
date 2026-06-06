# Label · Selector & Namespace

> 리소스를 **분류(label)** 하고 **격리(namespace)** 하는 방법. k8s가 수많은 오브젝트를 묶고 찾는 기반이다.

## Label & Selector

- **Label**: 오브젝트에 붙이는 key/value 태그. 식별·그룹핑·선택에 쓰인다.
- **Selector**: 라벨로 오브젝트를 **골라내는** 질의. Service가 Pod를 찾고, ReplicaSet이 Pod를 관리하는 것도 전부 selector 기반.

![라벨과 셀렉터](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_04_labels.svg)

> *출처: [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/) — 라벨로 오브젝트를 묶고, 셀렉터로 골라낸다.*

```yaml
metadata:
  labels:
    app: nginx
    tier: frontend
    env: prod
```

```bash
kubectl get pod -l app=nginx                 # equality
kubectl get pod -l 'env in (prod,staging)'   # set-based
kubectl get pod -l app=nginx,tier=frontend   # AND
kubectl get pod --show-labels                # 라벨 같이 보기
```

| selector 유형 | 예 |
|---|---|
| equality-based | `app=nginx`, `env!=dev` |
| set-based | `env in (prod,staging)`, `tier notin (cache)`, `app`(존재), `!app`(부재) |

> ⚠️ ReplicaSet/Deployment의 `spec.selector`와 `template.metadata.labels`는 **일치해야** 한다.

## Annotation

- 라벨과 비슷하지만 **선택(selector)에 쓰지 않는** 메타데이터. 도구·라이브러리용 부가 정보(빌드 정보, 변경 이유, ingress 설정 등).
- 식별/그룹핑 → **label**, 비식별 부가정보 → **annotation**.

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "update image to 1.28"
    nginx.ingress.kubernetes.io/rewrite-target: /
```

## Namespace

하나의 클러스터를 **논리적으로 나누는 칸막이**. 이름 충돌 방지, 권한·쿼터 경계.

### 기본 네임스페이스

| 네임스페이스 | 용도 |
|---|---|
| `default` | 별도 지정 안 하면 여기 |
| `kube-system` | 클러스터 컴포넌트(coredns, kube-proxy 등) |
| `kube-public` | 모두 읽기 가능한 공개용 |
| `kube-node-lease` | 노드 heartbeat(lease) |

> ⚠️ 이 **4개는 시스템(예약) 네임스페이스 — 삭제하지 말 것.** 특히 `default`엔 apiserver로 가는 `kubernetes` Service와 각 ns에 자동 생성되는 `default` ServiceAccount가 얽혀 있어, 지우면 `Terminating`에 걸리거나 클러스터가 이상해질 수 있다. `delete ns`로 통째 정리하는 건 **직접 만든 실습용 ns에만** 적용한다.

```bash
kubectl get ns
kubectl create namespace dev
kubectl get pods -n dev                # 특정 네임스페이스
kubectl get pods -A                    # 전체 네임스페이스
kubectl config set-context --current --namespace=dev   # 기본 ns 변경
```

### namespaced vs cluster-scoped

- **namespaced**(네임스페이스에 속함): Pod, Deployment, Service, ConfigMap, Secret, PVC, Role…
- **cluster-scoped**(클러스터 전역): Node, PersistentVolume, Namespace, ClusterRole, StorageClass…

```bash
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

### 네임스페이스 간 통신 (DNS)

서비스 DNS 이름에 네임스페이스가 들어간다:
```
<service>.<namespace>.svc.cluster.local
예) db.dev.svc.cluster.local
```
- 같은 네임스페이스면 `<service>`만으로도 접근. (DNS 상세 → [`04_services-networking`](../04_services-networking/))

> 네임스페이스 단위 자원 제한(`ResourceQuota`)·기본 한도(`LimitRange`)는 [`03_workloads-scheduling`](../03_workloads-scheduling/)와 연결.

## 시험·실무 팁

- 실습은 **전용 네임스페이스**를 만들어 하고 끝나면 `kubectl delete ns <name>`으로 통째로 정리하면 깔끔.
- 문제마다 네임스페이스가 다를 수 있으니 `-n`을 빼먹지 말 것.
- 라벨로 대량 작업: `kubectl delete pod -l app=nginx`.

## 참고

- [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
