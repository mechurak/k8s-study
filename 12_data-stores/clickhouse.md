# ClickHouse — 컬럼 기반 OLAP DB

> 대량의 이벤트/로그/메트릭을 빠르게 집계·분석하는 **OLAP 데이터베이스**. Langfuse(LLM 관측)의 분석 백엔드, 사내 데이터 분석 저장소로 많이 쓰인다.
>
> OLTP(Postgres)와의 역할 대비, Langfuse 데이터 스택에서의 위치는 [README](./README.md) 참고. 실습은 [practice-clickhouse.md](./practice-clickhouse.md).

## 1. 왜 빠른가

- **컬럼 기반 저장(columnar)**: 컬럼별로 따로 저장 → 집계에 필요한 컬럼만 디스크에서 읽음(I/O 절감). 같은 컬럼은 값이 비슷해 **압축률**도 매우 높다.
- **벡터화 실행**: 데이터를 블록 단위로 묶어 CPU SIMD로 한 번에 처리.
- **MergeTree 엔진**: ClickHouse 기본 테이블 엔진 패밀리. 데이터를 정렬된 조각(part)으로 쓰고 백그라운드에서 병합(merge) → **대량 INSERT + 빠른 범위 스캔**에 최적.

> ⚠️ 반대로 **단건 UPDATE/DELETE, 강한 트랜잭션(ACID), 빈번한 작은 INSERT** 는 약하다. 이런 워크로드는 PostgreSQL 몫 → [postgres.md](./postgres.md).

## 2. 핵심 포트

| 포트 | 용도 | 누가 쓰나 |
|---|---|---|
| **8123** | HTTP 인터페이스 (REST·JDBC) | **DBeaver**, `curl`, HTTP 클라이언트 |
| **9000** | 네이티브 TCP 프로토콜 | `clickhouse-client`, **Grafana**(네이티브), 고성능 앱 |
| 9009 | 노드 간 복제(interserver) | 멀티 노드 클러스터 (단일 노드엔 불필요) |
| 8443 / 9440 | 위 8123/9000의 TLS 버전 | 프로덕션 |

헬스체크는 HTTP `GET /ping` → `Ok.` 응답. (StatefulSet probe가 이걸 씀)

## 3. k8s 배포 — 왜 StatefulSet인가

DB는 **(1) 안정적인 네트워크 식별자**와 **(2) 자기 전용 디스크**가 필요하다. Deployment의 파드는 이름이 매번 바뀌고 디스크를 공유/유실할 수 있어 부적합. StatefulSet은:

- 파드 이름이 `clickhouse-0`, `clickhouse-1`… 으로 **고정**되고
- `volumeClaimTemplates` 로 **파드마다 전용 PVC**(`data-clickhouse-0`)를 자동 생성하며
- **Headless Service**(`clusterIP: None`)와 짝지어 파드별 DNS(`clickhouse-0.clickhouse.<ns>.svc`)를 부여한다. ([Headless 개념 → `04_services-networking`](../04_services-networking/#headless-service--파드를-콕-집어-부른다))

> 개념 자체는 — **StatefulSet 워크로드**는 [`03_workloads-scheduling`](../03_workloads-scheduling/), **PV/PVC·StorageClass 스토리지**는 [`05_storage`](../05_storage/) 참고. 여기선 그걸 ClickHouse로 **실전 응용**한다.

**배포 옵션 비교**:

| 방식 | 언제 | 비고 |
|---|---|---|
| **직접 StatefulSet** (이 폴더 실습) | 학습 — k8s 프리미티브를 손으로 이해 | 단일 노드. 복제/샤딩은 직접 구성 |
| **Altinity clickhouse-operator** | 실무 — 클러스터/복제/스케일 | `ClickHouseInstallation` CRD로 선언. 사실상 표준 운영 도구 |
| **Bitnami Helm chart** | 빠른 기동 | values로 복제/인증 설정 |

## 4. DBeaver · Grafana 연결 (요점)

- **DBeaver**: ClickHouse를 JDBC(HTTP **8123**)로 연결. Host `localhost`, Port `8123`, DB `default`. (port-forward 필요)
- **Grafana**: 공식 `grafana-clickhouse-datasource`(ClickHouse Inc.+Grafana Labs) 또는 Altinity `vertamedia-clickhouse-datasource`(구 Vertamedia, 다운로드 1,600만+) — 둘 다 무료 OSS. 네이티브(9000) 또는 HTTP(8123)로 연결.

구체적인 접속 파라미터·단계는 [practice-clickhouse.md](./practice-clickhouse.md) 4·5절.

## 참고
- [ClickHouse 공식 문서](https://clickhouse.com/docs) · [Docker로 설치](https://clickhouse.com/docs/en/install#docker) · [이미지 태그(Docker Hub)](https://hub.docker.com/r/clickhouse/clickhouse-server/)
- [MergeTree 엔진](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree)
- [Altinity clickhouse-operator](https://github.com/Altinity/clickhouse-operator)
- [Grafana ClickHouse 데이터소스(공식)](https://grafana.com/grafana/plugins/grafana-clickhouse-datasource/) · [Altinity 플러그인](https://grafana.com/grafana/plugins/vertamedia-clickhouse-datasource/)
