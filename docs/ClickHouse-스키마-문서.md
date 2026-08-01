# ClickHouse 스키마 문서

> Kafka 뒤에 ClickHouse를 두는 이유, 테이블을 왜 이렇게 나누고 이렇게 정렬하는지를
> OTel ClickHouse exporter 표준과 비교하며 정리한다. apm-infra의 실제 DDL(`clickhouse/init/`)을
> 기준으로 서술한다.

---

## 1. 파이프라인에서 ClickHouse의 위치

```
OTel Java Agent → OTel Collector → Kafka → 컨슈머 (Spring Boot) → [ClickHouse]
                                    버퍼                            영속 저장소
```

Kafka는 retention 24시간짜리 버퍼고, 조회·집계는 ClickHouse가 담당한다. 컨슈머가
`ExportTraceServiceRequest`/`ExportMetricsServiceRequest`를 역직렬화해 아래 테이블에 배치 INSERT한다.
이 레포(apm-infra)는 컨테이너와 스키마까지만 만들고, 적재 로직은 컨슈머 앱 별도 이슈에서 구현한다.

## 2. 왜 정규화하지 않는가 — wide denormalized table

ClickHouse는 OLAP 컬럼 스토어라 관계형 DB와 설계 원칙이 반대다.

| 관계형 DB (MySQL) | ClickHouse |
|-------------------|------------|
| 정규화 + FK로 중복 제거 | 비정규화 — 조인 회피가 우선 |
| 행 단위 저장 | 컬럼 단위 저장 (필요한 컬럼만 스캔) |
| FK 제약을 DB가 강제 | 제약 없음 — 관계는 논리적 |

그래서 `otel_traces`는 span 한 줄에 resource 정보(`ServiceName`, `ResourceAttributes`)를 매번 복제해 넣는다.
service 테이블을 따로 두고 조인하지 않는다. 저장 공간은 늘지만 `LowCardinality`+`ZSTD` 압축으로 상쇄되고,
"트레이스 조회 = trace_id 필터 후 정렬"이라는 접근 패턴에서 조인 제거가 훨씬 이득이다.

## 3. otel_traces — span 1개 = 1행

핵심 컬럼과 관계:

- **TraceId**: 같은 요청의 모든 span을 묶는 키. 조회는 `WHERE TraceId = ?`
- **ParentSpanId → SpanId**: 같은 테이블 안의 self-reference. 이걸로 span 트리(콜스택 뷰)를 재구성. FK가 아니라 애플리케이션이 해석하는 논리적 관계
- **ServiceName**: resource 속성을 비정규화해 각 행에 복제
- **Events / Links**: `Nested` 타입 — span 내부 이벤트와 다른 트레이스 연결. ClickHouse가 내부적으로 `Events.Timestamp Array(...)`, `Events.Name Array(...)`처럼 병렬 배열로 펼쳐 저장

### 엔진 설계

```
ENGINE = MergeTree
PARTITION BY toDate(Timestamp)
ORDER BY (ServiceName, SpanName, toDateTime(Timestamp))
TTL toDateTime(Timestamp) + toIntervalHour(72)
```

- **ORDER BY**는 primary key 겸 sparse index. 조회 패턴이 "서비스별·이름별·시간순"이라 이 순서. `TraceId`는 여기 없음 — 무작위 값이라 정렬 키로 부적합
- **TraceId 단건 조회**는 그래서 `bloom_filter` skip index로 처리. `EXPLAIN indexes=1`에서 `idx_trace_id` 사용 확인됨
- **PARTITION BY toDate**: 날짜별 파티션 → TTL이 파티션 단위로 통째 삭제(`ttl_only_drop_parts=1`)돼 효율적
- **TTL 30일**: 주·월 단위 추이 조회가 목적. Kafka retention(24h)보다 길어 컨슈머 지연·재처리 여유도 확보된다

## 4. metrics — 타입마다 테이블 분리

메트릭은 타입별로 데이터 모양이 달라 한 테이블에 못 담는다. OTel 표준대로 타입별 테이블로 나눈다.

| 테이블 | 타입별 컬럼 | 실제 메트릭 (apm-demo) |
|--------|-----------|------------------------|
| `otel_metrics_gauge` | `Value` | jvm.cpu.recent_utilization |
| `otel_metrics_sum` | `Value`, `AggregationTemporality`, `IsMonotonic` | jvm.cpu.time, jvm.class.loaded, jvm.memory.used, jvm.thread.count (UpDownCounter도 sum으로 적재) |
| `otel_metrics_histogram` | `Count`, `Sum`, `BucketCounts`, `ExplicitBounds`, `Min`, `Max` | http.server.request.duration, jvm.gc.duration, db.client.connections.use_time |

3종만 만드는 이유: OTel 표준은 5종(gauge/sum/histogram/exp_histogram/summary)이지만, JVM+HTTP+Hikari 자동 계측이
이 3종만 내보낸다. exp_histogram·summary는 파이프라인에 흐르지 않아 제외했다. 필요해지면 테이블 추가로 확장.

엔진 설계는 traces와 유사하되 `ORDER BY (ServiceName, MetricName, Attributes, TimeUnix)` — 메트릭은 이름·라벨별
시계열 조회라 SpanName 대신 MetricName·Attributes가 정렬 키다. TraceId 단건 조회가 없어 bloom filter도 없다.

## 5. metrics ↔ traces 연결 — FK가 아니라 Exemplar

metrics 테이블과 traces 테이블을 잇는 길은 두 가지뿐이다.

1. **ServiceName + 시간**: "20:36에 apm-demo 응답시간이 튀었다(histogram) → 같은 시간대 느린 span 찾기(traces)"의 느슨한 상관관계
2. **Exemplars**: 각 metrics 테이블의 `Exemplars.TraceId`/`Exemplars.SpanId` 컬럼. histogram 데이터포인트에 "이 분포에 기여한 대표 샘플의 trace_id"가 담긴다. 이것이 메트릭 → 트레이스로 점프하는 유일한 직접 링크 — Grafana에서 레이턴시 스파이크 클릭 → 해당 트레이스 이동이 이걸로 동작

즉 이 스키마의 "관계선"은 traces 내부의 `ParentSpanId → SpanId` self-loop 하나와, metrics의 `Exemplars.TraceId → otel_traces.TraceId` 점선 하나뿐이다. 테이블 간 FK는 없다.

## 6. 초기화 동작

- `clickhouse/init/*.sql`은 `/docker-entrypoint-initdb.d`에 마운트돼 **볼륨이 빈 첫 기동에만** 파일명 순(01→02) 실행
- 스키마를 바꾸려면 `docker compose down -v`로 `clickhouse-data` 볼륨을 비우고 재기동해야 재실행됨 (MySQL 초기화와 동일한 제약)
