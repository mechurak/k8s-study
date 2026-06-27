# PostgreSQL — 범용 OLTP 관계형 DB

> 가장 널리 쓰이는 오픈소스 관계형 DB(RDBMS). 강한 트랜잭션(ACID)과 정합성이 필요한 **OLTP** 워크로드의 기본값. Langfuse의 메타데이터/트랜잭션 저장소이자, 우리 레포의 ArgoCD GitOps 실습 대상이다.
>
> OLAP(ClickHouse)와의 역할 대비, Langfuse 데이터 스택에서의 위치는 [README](./README.md) 참고.

## 1. 특징 — 무엇을 잘하나

- **행 기반(row) 저장 + B-tree 인덱스**: `WHERE id=42` 처럼 **특정 행을 콕 집어** 읽고 쓰는 데 강하다.
- **ACID 트랜잭션**: 약어 — **A**tomicity(원자성)·**C**onsistency(일관성)·**I**solation(격리성)·**D**urability(지속성). 여러 변경을 "전부 성공 아니면 전부 취소"로 묶는다.
- **MVCC**(**M**ulti-**V**ersion **C**oncurrency **C**ontrol, 다중 버전 동시성 제어): 읽기와 쓰기가 서로를 막지 않도록 행의 여러 버전을 유지. 동시 트랜잭션 처리량을 높인다.
- **풍부한 기능**: JSON/JSONB, 전문검색, 지리정보(PostGIS), 다양한 인덱스(GIN/GiST), 확장(extension) 생태계.

> ⚠️ 반대로 **수억 행을 스캔하는 대량 집계 분석**은 컬럼 기반 OLAP(ClickHouse)보다 느리다. 그래서 분석은 ClickHouse로 분리한다 → [clickhouse.md](./clickhouse.md).

## 2. 핵심 포트

| 포트 | 용도 |
|---|---|
| **5432** | PostgreSQL 기본 접속 포트 (`psql`·DBeaver·앱 모두 이걸로) |

헬스체크는 `pg_isready -U <user> -d <db>` 로 한다(연결 가능 여부 확인). StatefulSet의 readiness/liveness probe가 이 명령을 쓴다.

## 3. k8s 배포 — StatefulSet

ClickHouse와 똑같은 이유로 **StatefulSet + `volumeClaimTemplates`** 로 띄운다. DB는 안정적 이름(`postgres-0`)과 전용 디스크가 필요하기 때문(개념은 [`05_storage`](../05_storage/)). 포인트 두 가지:

- **`PGDATA` 하위 디렉터리 지정**: 볼륨 루트에 `lost+found` 등이 생기면 `initdb`가 실패할 수 있어, 데이터 경로를 `/var/lib/postgresql/data/pgdata` 처럼 **하위 폴더**로 둔다.
- **probe는 `pg_isready` exec**: HTTP가 아니라 명령 실행으로 준비 상태를 본다(ClickHouse의 HTTP `/ping`과 대비).

**운영 옵션 비교**(실무):

| 방식 | 언제 | 비고 |
|---|---|---|
| **직접 StatefulSet** | 학습·소규모 | 단일 인스턴스. HA·백업·페일오버는 직접 |
| **CloudNativePG** (operator) | 실무 — k8s 네이티브 HA | `Cluster` CRD. 자동 페일오버·PITR 백업. 현재 가장 권장 |
| **Zalando postgres-operator** | 실무 — 오래된 표준 | Patroni 기반 HA |
| **Bitnami Helm chart** | 빠른 기동 | 복제(read replica) 옵션 |
| **AWS RDS / Aurora** | 실무(EKS) — 관리형 | DB를 클러스터 밖 관리형으로 분리 ([`09_aws-eks`](../09_aws-eks/)) |

> 💡 실무 감각: **상태가 있는 DB는 가능하면 관리형(RDS)이나 검증된 operator로** 운영하고, 직접 StatefulSet은 학습·내부도구 수준에서 쓰는 게 보통이다.

## 4. 우리 레포에서의 위치

PostgreSQL은 [`10_ecosystem-gitops`](../10_ecosystem-gitops/)에서 **ArgoCD GitOps 실습의 배포 대상**으로 이미 운영된다. (Postgres 자체보다 GitOps 워크플로를 배우는 소재)

| 파일 | 내용 |
|---|---|
| [`10_ecosystem-gitops/manifests/postgres/statefulset.yaml`](../10_ecosystem-gitops/manifests/postgres/statefulset.yaml) | `postgres:17` 단일 인스턴스, PVC 영속 (user `app` / db `appdb`) |
| `…/postgres/service.yaml` · `secret.yaml` | Headless Service + 비밀번호 |
| `…/argocd-apps/postgres-app.yaml` | 위를 Git에서 동기화하는 ArgoCD `Application` |
| [`10_ecosystem-gitops/recipes.md`](../10_ecosystem-gitops/recipes.md) | "DB를 처음부터 다시 만들기"(PVC 삭제 후 재동기화) 레시피 |

→ 그래서 이 폴더(12)에는 별도 Postgres 매니페스트를 두지 않고 **10번 것을 재사용**한다. Langfuse 풀스택을 구성할 땐 여기서 Postgres가 **메타데이터 스토어**로 들어간다([README](./README.md)의 데이터 스택 그림).

## 5. 접속 — psql · DBeaver

**클러스터 안에서 `psql`**(가장 흔한 진단):

```bash
# ns·파드 이름은 환경에 맞게 (GitOps 실습 기준: ns=database, pod=postgres-0)
kubectl -n database exec -it postgres-0 -- psql -U app -d appdb -c '\dt'
```

**DBeaver로 연결**(로컬 GUI) — 5432를 포워딩 후 붙는다:

```bash
kubectl -n database port-forward svc/postgres 5432:5432
```

| 항목 | 값 |
|---|---|
| Host / Port | `localhost` / `5432` |
| Database | `appdb` |
| Username | `app` |
| Password | (Secret `postgres`의 `POSTGRES_PASSWORD`) |

## 6. 백업 (개념)

- **논리 백업**: `pg_dump`(단일 DB) / `pg_dumpall`(전체) → SQL 덤프. 소규모·이식에 적합.
- **물리 백업 + PITR**(**P**oint-**I**n-**T**ime **R**ecovery, 특정 시점 복구): 베이스 백업 + WAL(**W**rite-**A**head **L**og) 아카이브로 임의 시점 복구. operator(CloudNativePG 등)가 자동화해 준다.

```bash
# 파드 안에서 논리 백업 한 번 떠보기
kubectl -n database exec -it postgres-0 -- pg_dump -U app appdb > appdb.sql
```

## 참고
- [PostgreSQL 공식 문서](https://www.postgresql.org/docs/) · [트랜잭션·MVCC](https://www.postgresql.org/docs/current/mvcc.html) · [pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html)
- [CloudNativePG (operator)](https://cloudnative-pg.io/) · [Zalando postgres-operator](https://github.com/zalando/postgres-operator)
- 관련: [`10_ecosystem-gitops`](../10_ecosystem-gitops/)(Postgres GitOps 배포) · [`05_storage`](../05_storage/)(StatefulSet 스토리지) · [`09_aws-eks`](../09_aws-eks/)(관리형 RDS)
