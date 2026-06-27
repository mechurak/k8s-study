# ClickHouse 실습 — kind에 단일 노드 띄우고 DBeaver·Grafana 연결

> 목표: ClickHouse를 **StatefulSet으로 직접 배포**하고, `clickhouse-client`·**DBeaver**로 쿼리, **Grafana**로 시각화까지. OLAP가 왜 빠른지 대량 INSERT/집계로 체감한다.
>
> 개념은 [clickhouse.md](./clickhouse.md) 참고. 클러스터 준비는 [`01_lab-environment/kind.md`](../01_lab-environment/kind.md).

전제: kind 클러스터가 떠 있고 `kubectl`이 그 클러스터를 가리킨다.

```bash
kubectl cluster-info        # 클러스터 연결 확인
```

---

## 1. 배포

전용 네임스페이스를 만들고 매니페스트 3종을 적용한다.

```bash
kubectl create namespace data
kubectl -n data apply -f manifests/        # secret → statefulset → service 한 번에
```

> 🔎 `kubectl apply -f <dir>` 는 디렉터리 안 YAML을 전부 적용한다. 개별로 적용하려면 `-f manifests/clickhouse-statefulset.yaml`.

생성 과정을 지켜본다:

```bash
kubectl -n data get pods -w        # clickhouse-0 이 Running·Ready 될 때까지 (Ctrl+C로 종료)
```

> 🔎 **관찰 포인트**
> - 파드 이름이 `clickhouse-0` — StatefulSet은 순번이 붙는다(Deployment의 랜덤 해시와 대비).
> - 처음엔 이미지 pull로 `ContainerCreating`이 좀 걸린다. `READY 1/1`이 되면 `/ping` probe 통과한 것.

---

## 2. 만들어진 오브젝트 관찰

```bash
kubectl -n data get statefulset,pod,svc,pvc
```

> 🔎 **관찰 포인트**
> - **PVC `data-clickhouse-0`** 가 자동 생성됨 — `volumeClaimTemplates`가 파드마다 전용 볼륨을 만든 결과.
> - Service `clickhouse` 의 `CLUSTER-IP` 가 **`None`**(headless). 그래서 파드별 DNS로 접근한다.
> - PVC가 바인딩한 StorageClass 확인: `kubectl -n data get pvc -o wide` → kind 기본 `standard`(local-path).

파드별 DNS가 사는지 확인(클러스터 내부 DNS):

```bash
kubectl -n data run dns-test --rm -it --image=busybox:1.36 --restart=Never -- \
  nslookup clickhouse-0.clickhouse.data.svc.cluster.local
```

---

## 3. clickhouse-client로 쿼리 (파드 안에서)

가장 흔한 접근 — 파드 안의 `clickhouse-client`로 바로 붙는다.

```bash
kubectl -n data exec -it clickhouse-0 -- clickhouse-client --password clickhouse123
```

프롬프트(`clickhouse-0 :)`)가 뜨면 아래를 입력:

```sql
SELECT version();
SHOW DATABASES;
```

### 테이블 만들고 대량 데이터 넣어 OLAP 체감

```sql
-- MergeTree: ClickHouse 기본 엔진. ORDER BY로 정렬 키 지정
CREATE TABLE default.events
(
    ts       DateTime,
    user_id  UInt32,
    action   LowCardinality(String),   -- 값 종류가 적은 문자열 → 사전 인코딩으로 초압축
    cost     Float64
)
ENGINE = MergeTree
ORDER BY (ts, user_id);

-- 1천만 행을 한 방에 생성·삽입 (numbers()로 생성)
INSERT INTO default.events
SELECT
    now() - toIntervalSecond(number % 2592000)          AS ts,         -- 최근 30일에 분포
    (number % 100000)                                   AS user_id,
    ['view','click','purchase','signup'][1 + number % 4] AS action,
    round(rand() % 10000 / 100, 2)                      AS cost
FROM numbers(10000000);

SELECT count() FROM default.events;        -- 1천만
```

집계 질의를 돌려본다:

```sql
-- 액션별 건수·매출 (수백만 행을 순식간에 집계)
SELECT action, count() AS cnt, round(sum(cost)) AS revenue
FROM default.events
GROUP BY action
ORDER BY revenue DESC;

-- 일자별 추이
SELECT toDate(ts) AS day, count() AS cnt
FROM default.events
GROUP BY day
ORDER BY day;
```

> 🔎 **관찰 포인트**
> - 1천만 행 집계가 보통 **수십~수백 ms**. 컬럼 기반 + 벡터화의 힘.
> - 끝에 `FORMAT JSON` 이나 `\G`(세로 출력) 같은 포맷도 시도해 보자.
> - 압축 효과 확인: `SELECT formatReadableSize(sum(data_compressed_bytes)) AS comp, formatReadableSize(sum(data_uncompressed_bytes)) AS raw FROM system.columns WHERE table='events';`

나가기: `exit`.

---

## 4. DBeaver로 연결 (로컬 GUI)

> **DBeaver**("디비버"로 읽는다 — DB + beaver(비버), 로고도 비버다)**는 내 PC(로컬 호스트)에 설치하는 데스크톱 GUI 앱**이다. 클러스터 안에 띄우는 게 아니다 — DBeaver는 *클라이언트*, ClickHouse는 *서버*. "서버는 클러스터 안, GUI 클라이언트는 내 PC"가 기본 패턴이다. (데스크톱 GUI는 파드에 넣어도 화면을 볼 수 없다. 웹으로 쓰는 서버판 [CloudBeaver](https://github.com/dbeaver/cloudbeaver)라면 클러스터 안에 띄울 만하지만 지금은 오버킬.)
>
> 설치(Community Edition, 무료):
> - **macOS**: `brew install --cask dbeaver-community`
> - **Ubuntu**: `sudo snap install dbeaver-ce` (또는 [dbeaver.io/download](https://dbeaver.io/download/)에서 `.deb`)
>
> DBeaver 없이 3절의 `clickhouse-client`만으로도 모든 쿼리는 가능하다 — GUI 탐색·결과 그리드가 편해서 쓰는 선택지일 뿐.

ClickHouse는 클러스터 **안**에 있어 로컬에서 바로 못 붙는다. HTTP 포트(8123)를 로컬로 포워딩해 터널을 뚫는다(클러스터 안 8123 → 내 PC의 `localhost:8123`).

```bash
kubectl -n data port-forward svc/clickhouse 8123:8123
# (이 터미널은 켜둔 채로 둔다)
```

DBeaver에서 **새 연결 → ClickHouse**:

| 항목 | 값 |
|---|---|
| Host | `localhost` |
| Port | `8123` |
| Database | `default` |
| Username | `default` |
| Password | `clickhouse123` |

> 🔎 처음 연결 시 DBeaver가 ClickHouse JDBC 드라이버를 자동 다운로드한다. 연결 후 `default.events` 테이블이 보이고, 3절의 집계 SQL을 GUI에서 그대로 돌릴 수 있다.
>
> 📌 **JDBC**(Java Database Connectivity): 자바 프로그램이 DB에 접속해 SQL을 주고받는 표준 규약. **JDBC 드라이버**는 그 규약을 특정 DB(여기선 ClickHouse) 전용 프로토콜로 번역해주는 부품으로, DB마다 따로 있다. DBeaver가 자바 기반이라 필요하며, 처음 연결 때 알아서 받는다. (반면 3절의 `clickhouse-client`는 자바가 아니라 native 프로토콜(9000)로 붙어 JDBC가 필요 없다.)
>
> 💡 드라이버 설정에서 `ssl=false` 가 기본. 사내/원격 ClickHouse는 보통 8443(HTTPS)+SSL on.

---

## 5. Grafana로 시각화 (선택)

이미 Grafana가 있으면 데이터소스만 추가하면 된다. 없으면 kind에 간단히 띄운다:

```bash
# 빠른 기동용 (학습용 단발 실행)
kubectl -n data create deployment grafana --image=grafana/grafana:11.2.0
kubectl -n data expose deployment grafana --port=3000
kubectl -n data port-forward deploy/grafana 3000:3000
# 브라우저 http://localhost:3000  (admin / admin)
```

Grafana에서:

1. **Connections → Add new connection → "ClickHouse"** 검색 → 공식 `grafana-clickhouse-datasource` 설치(필요 시).
2. 데이터소스 설정:
   - Server address: `clickhouse.data.svc.cluster.local` (같은 클러스터 내부 DNS)
   - Port: `9000` (Native) 또는 `8123` (HTTP) — Protocol을 맞춘다
   - Username `default` / Password `clickhouse123`
3. **Explore** 또는 새 대시보드에서 쿼리:
   ```sql
   SELECT toStartOfHour(ts) AS t, count() AS c
   FROM default.events GROUP BY t ORDER BY t
   ```
   시간 컬럼을 `t`로 두면 시계열 패널로 그려진다.

> 🔎 Grafana 파드는 클러스터 **안**이라 ClickHouse Service DNS로 바로 붙는다(port-forward 불필요). DBeaver는 클러스터 **밖**이라 port-forward가 필요했던 것과 대비.

---

## 6. 정리(cleanup)

```bash
kubectl delete namespace data        # 모든 오브젝트 삭제
```

> ⚠️ **PVC는 네임스페이스 삭제로 함께 지워진다 → 데이터도 사라진다.** 데이터를 남기려면 네임스페이스를 지우지 말고 StatefulSet만 `kubectl -n data delete statefulset clickhouse` 로 지운다(PVC는 남음 → 재배포 시 데이터 유지).

---

## 🧪 과제

1. **데이터 영속성 확인**: StatefulSet만 삭제 → 다시 `apply` → `clickhouse-0` 재생성 후 `SELECT count() FROM default.events` 가 그대로인가? (PVC가 살아있어 데이터 유지) 반대로 PVC까지 지우면?
2. **ConfigMap으로 설정 주입**: `max_concurrent_queries` 나 `display_name` 을 바꾸는 `config.d/*.xml` 을 ConfigMap으로 만들어 `/etc/clickhouse-server/config.d/` 에 마운트해 보자. (힌트: [`03_workloads-scheduling`](../03_workloads-scheduling/)의 ConfigMap 마운트)
3. **probe 깨보기**: `livenessProbe` 포트를 일부러 9999로 바꿔 apply → 파드가 계속 재시작(`CrashLoopBackOff` 비슷)하는 것 관찰 → 원복.
4. **압축 비교**: `action` 컬럼을 `LowCardinality(String)` 대신 그냥 `String` 으로 둔 테이블을 따로 만들어, `system.columns`의 압축 크기를 비교해 보자.
5. **다음 단계 — Langfuse**: ClickHouse + Postgres + Redis + MinIO를 묶는 [Langfuse 셀프호스팅](https://langfuse.com/self-hosting)으로 확장. 여기 ClickHouse가 trace 저장소로 들어간다. ([README](./README.md)의 데이터 스택 그림 참고)
