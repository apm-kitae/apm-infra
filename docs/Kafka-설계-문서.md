# Kafka 설계 문서

> Collector와 컨슈머 사이에 Kafka를 두는 이유, 토픽·파티션·컨슈머 그룹 설계 기준을
> 기업 기술 블로그 사례와 비교하며 정리한다. apm-infra의 실제 설정을 기준으로 서술한다.

---

## 1. 파이프라인에서 Kafka의 위치

```
OTel Java Agent → OTel Collector → [Kafka] → 컨슈머 (Spring Boot) → ClickHouse
                                    버퍼
```

텔레메트리 파이프라인을 공개한 기업들(Uber, Netflix, LinkedIn, Cloudflare, Datadog,
New Relic, Grafana, 카카오페이증권, 네이버 NELO2, 카카오 KEMI, 토스증권)은 전부
Kafka를 **수집기와 저장소 사이의 버퍼**로 둔다. 이유는 3가지로 수렴한다.

| 이유 | 사례 |
|------|------|
| **저장소 장애 흡수** | Cloudflare — 컨슈머 전면 장애를 8시간까지 데이터 유실 없이 버팀. 저장소가 죽어도 Kafka에 쌓이고 복구 후 따라잡음 |
| **유입 폭증 흡수** | 카카오페이증권 — 09:00 국내장 개장 스파이크(피크 초당 83만 건)를 Kafka에 받아두고 컨슈머 수로 처리 속도 조절 |
| **fan-out (멀티 컨슘)** | 같은 데이터를 여러 시스템이 각자 소비 (4절) |

**반례**: ClickHouse 공식 문서는 "ClickHouse는 초당 수백만 row 삽입이 가능해 일반
규모에선 Kafka가 불필요한 복잡도"라고 명시하고, 우아한형제들 전사 로그(피크 초당
100만 레코드)는 Kafka 없이 Fluentd 디스크 버퍼 + Loki로 운영한다. 즉 Kafka는 필수가
아니라 **재생(replay)·전달 보증·장애 영속성이 필요할 때 넣는 선택**이다. 이 프로젝트는
컨슈머를 직접 구현해 파이프라인을 학습하는 목적이므로 채택한다.

---

## 2. 토픽 설계 — 무엇 기준으로 쪼개는가

지배적 패턴은 **신호(데이터 종류)별 분리**다.

| 기업 | 토픽 분리 기준 |
|------|---------------|
| Netflix Keystone | 이벤트 타입당 토픽 1개 — 싱크 장애를 토픽 단위로 격리 |
| LinkedIn | metrics / logging / tracking을 **클러스터 수준**에서 분리 |
| 카카오페이증권 | 서비스별 300개 토픽 → **로그 타입 3종 × 처리 레벨 3종 = 18개로 통합** ("단일 토픽 소비가 여러 토픽 소비보다 6배 빠르다") |
| Datadog / New Relic | 신호 분리 + 테넌트(고객) 차원 추가 |

카카오페이증권 사례의 교훈: 토픽을 잘게 쪼개면(서비스별 300개) 컨슈머 관리가 폭발하고
처리량도 떨어진다. 쪼개는 단위는 **"이 데이터가 밀려도 저 데이터는 밀리면 안 된다"의
경계(격리 단위)** 만큼이다.

### apm-infra의 설계

| 토픽 | 파티션 | 메시지 키 | 값 |
|------|--------|----------|-----|
| `traces` | 3 | trace_id | OTLP Protobuf (`ExportTraceServiceRequest`) |
| `metrics` | 3 | 없음 (라운드로빈) | OTLP Protobuf (`ExportMetricsServiceRequest`) |

- 격리 근거: 메트릭이 폭증해도 트레이스 소비가 밀리지 않는다. 컨슈머의 파싱 타입
  (`ExportTraceServiceRequest` vs `ExportMetricsServiceRequest`)도 신호별로 다르다
- logs는 토픽 미전송 (debug exporter로만 출력). 필요 시 토픽 추가로 확장

---

## 3. 파티셔닝 — 병렬성과 순서의 트레이드오프

파티션은 토픽 내부의 병렬 처리 단위이고, **순서는 파티션 안에서만 보장**된다.
따라서 파티션 키 선택 = "무엇의 순서를 지킬 것인가"의 결정이다.

| 기업 | 파티션 키 | 지키려는 것 |
|------|----------|------------|
| Cloudflare | host + service | 같은 머신·서비스 로그의 순서 |
| New Relic | 고객 account (일부 구간 랜덤) | 계정별 데이터 지역성 vs 부하 균등 |
| Datadog | tenant_id 해싱 → shard | 테넌트 단위 처리 |
| Grafana Mimir | time series 해싱 | 시리즈 단위 샤딩 |

### traces: `partition_traces_by_id: true` (키 = trace_id)

- 같은 트레이스의 모든 span이 같은 파티션 → 같은 컨슈머 인스턴스에 도착
- 컨슈머의 트레이스 단위 조립·집계가 단순해지고, tail-based sampling(트레이스 전체를
  보고 버릴지 결정)을 하려면 사실상 필수 패턴
- 단점: 비정상적으로 큰 트레이스가 한 파티션에 몰리는 핫 파티션 위험 (데모 규모 무관)

### metrics: 키 없음 (라운드로빈)

메트릭에는 트레이스 같은 묶음 개념이 없어 균등 분산이 적합하다.

### 파티션 수는 미리 크게

카카오페이증권은 토픽당 150 파티션에 컨슈머 15~50개로 운영한다. 이유: **밀린 백로그는
컨슈머 증설로만 따라잡을 수 있고, 파티션을 나중에 늘려도 이미 쌓인 데이터는 재분산되지
않는다.** 그래서 "파티션 수 ≥ 미래의 최대 컨슈머 수"로 미리 잡는다.
apm-infra는 3개 — 컨슈머 1개로 시작해도 인스턴스 3개까지의 수평 확장 구조를 보여주는
최소 구성이다.

---

## 4. 컨슈머 그룹과 fan-out (멀티 컨슘)

규칙은 두 줄이다.

- **같은 컨슈머 그룹 안**: 파티션을 나눠 갖는다 → 분업 (처리량 확장)
- **다른 컨슈머 그룹끼리**: 같은 데이터를 각자 처음부터 전부 받는다 → fan-out (용도 확장)

그룹마다 오프셋("어디까지 읽었는지")을 따로 기록하기 때문에 가능하다.

| 기업 | fan-out 사례 |
|------|-------------|
| Cloudflare | 같은 로그 토픽을 Logstash(→Elasticsearch 검색)와 inserter(→ClickHouse 분석)가 각자의 그룹으로 동시 소비 |
| LINE | 전사 데이터 허브 — "하나의 토픽에 여러 컨슈머가 각각 다른 목적으로 존재" (일 2,500억 건, 단일 클러스터 세계 최대 규모) |
| 카카오 KEMI | 로그를 Kafka와 Hadoop에 이원 적재 → 실시간 처리와 배치 색인 분리 |
| Grafana Mimir | 여러 zone의 ingester가 같은 레코드를 각각 소비 (읽기 HA) |

### apm 파이프라인 적용

컨슈머 앱은 `apm-consumer-traces` 같은 그룹 ID로 소비한다. 이후 "ClickHouse 적재와
별개로 실시간 알림 검사기를 붙이고 싶다"면 **컨슈머 그룹 하나 추가**로 끝난다 —
Collector도 기존 컨슈머도 변경 없음. 이것이 Kafka를 중간에 둔 구조의 핵심 확장 이점이다.

---

## 5. 메시지 포맷 — Kafka에 무엇이 저장되는가

기업별 포맷은 제각각이다(LinkedIn·Netflix Avro, Cloudflare Cap'n Proto, Uber 로그 JSON).
방향을 보여주는 사례는 카카오페이증권: **JSON → OTLP Protobuf 전환으로 용량 40~60% 절감,
여러 건을 메시지 1개로 배치해 처리량 18배**. OTel 생태계의 표준도 OTLP Protobuf다.

apm-infra의 토픽에 들어가는 메시지 1개의 구조:

```
[Kafka 메시지]
├── key   : trace_id 바이트 (partition_traces_by_id)
└── value : ExportTraceServiceRequest 를 Protobuf 직렬화한 바이트
            └── ResourceSpans[]          ← debug 로그의 3층 구조 그대로
                ├── Resource (service.name, ...)
                └── ScopeSpans[] → Span[] (traceId, parentId, name, kind, ...)
```

**debug 로그로 본 구조가 기계용 직렬화로 바뀌었을 뿐 내용이 동일하다** —
[OTel Log 읽기 문서](./OTel-Log-읽기-문서.md)의 구조 이해가 그대로 컨슈머 파싱 지식이
된다. Collector의 batch processor가 여러 span을 한 요청으로 묶어 보내므로 "N건 배치 =
메시지 1개"도 이미 적용된 상태다.

### 컨슈머의 역직렬화 (다음 단계)

```xml
<dependency>
    <groupId>io.opentelemetry.proto</groupId>
    <artifactId>opentelemetry-proto</artifactId>
    <version>1.9.0-alpha</version>  <!-- 안정 릴리스에도 -alpha 접미사 유지 정책 -->
</dependency>
```

```java
// Kafka 컨슈머는 ByteArrayDeserializer 사용
ExportTraceServiceRequest request = ExportTraceServiceRequest.parseFrom(record.value());
for (ResourceSpans rs : request.getResourceSpansList()) {
    // Resource attributes → service_name
    // ScopeSpans → Span 순회 → 컬럼 추출 → ClickHouse 배치 INSERT
}
```

metrics는 같은 패턴으로 `ExportMetricsServiceRequest.parseFrom(record.value())`.

---

## 6. Retention — 버퍼니까 짧게

공개된 수치는 전부 짧다: Netflix 4~6시간, Cloudflare 버퍼 8시간, New Relic 쿼리 토픽
1시간, OTel+ClickHouse 파이프라인 사례 24~72시간. 기준은 **"저장소 최대 장애 허용 시간 +
백로그를 따라잡는 시간"** 이다. 장기 보관은 ClickHouse/MinIO의 역할이지 Kafka의 역할이
아니다.

apm-infra: `KAFKA_LOG_RETENTION_HOURS: 24` — 위 범위의 표준값.

---

## 7. kafkaexporter 설정 근거 (contrib v0.156.0)

[kafkaexporter v0.156.0 README](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.156.0/exporter/kafkaexporter/README.md) 기준.

- top-level `topic` / `encoding`은 제거됨 → **신호별 블록**(`traces::topic`,
  `metrics::encoding`)으로만 설정 (기본 토픽명: `otlp_spans` / `otlp_metrics` / `otlp_logs`)
- `partition_traces_by_id` 등 partition 옵션 4종은 **top-level** 위치
- `protocol_version`은 선택 (기본 2.1.0)
- `encoding` 기본값 `otlp_proto`

```yaml
# otel-collector/config.yaml 발췌
exporters:
  kafka:
    brokers:
      - kafka:29092
    partition_traces_by_id: true
    traces:
      topic: traces
      encoding: otlp_proto
    metrics:
      topic: metrics
      encoding: otlp_proto
```

---

## 8. 설계 요약 — 실무 패턴 대비

| 항목 | 실무 패턴 | apm-infra |
|------|----------|-----------|
| Kafka 위치 | 수집기 ↔ 저장소 버퍼 | Collector ↔ 컨슈머 버퍼 — 동일 |
| 토픽 | 신호별 분리 | `traces`, `metrics` 2개 (logs 미전송) |
| 파티션 키 | 순서를 지킬 단위로 선택 | traces=trace_id, metrics=라운드로빈 |
| 파티션 수 | 컨슈머 최대치보다 크게 (예: 150) | 3 — 수평 확장 시연용 최소 |
| 포맷 | Avro/Protobuf 등 바이너리 | OTLP Protobuf (카카오페이증권과 동일) |
| retention | 1h~72h | 24h |
| 복제 | RF=3 + min.insync.replicas=2 | RF=1 (단일 브로커 데모) |
| 클러스터 | 브로커 3대+ (KRaft/ZK) | KRaft 단일 노드 |

구조 자체는 카카오페이증권 Pallas v2와 동일한 구성이다:
**OTel Collector → Kafka(신호별 토픽) → 컨슈머 → ClickHouse**.
실무에서 규모 때문에 추가되는 요소(복제, zstd 압축, 수십 개 파티션, 브로커 다중화)만
데모 규모로 축소했다.

---

## 9. 초기 설정값 가이드 — 실무 권장 vs apm-infra

### 브로커·토픽

| 설정 | Kafka 기본값 | 실무 시작 권장 | apm-infra |
|------|------------|---------------|-----------|
| 파티션 수 | 1 | 3~6 (Confluent Cloud 기본 6), 이후 `max(t/p, t/c)` 공식으로 산정 | 3 |
| replication.factor | 1 | 프로덕션 3 | 1 (단일 브로커) |
| min.insync.replicas | 1 | 프로덕션 2 (`acks=all`과 세트) | 1 (단일 브로커) |
| retention | 7일 | 버퍼 용도 1~7일 + `retention.bytes` 병행(디스크 안전장치) | 24시간 |
| **segment(roll) 주기** | **7일** | **retention 이하로** — retention 삭제는 닫힌 세그먼트에만 적용되므로, 롤링이 retention보다 길면 만료 데이터가 삭제되지 않음 (New Relic 실측: 1시간 TTL 토픽에 기본 7일 세그먼트 → 만료 데이터 잔류) | `log.roll.ms=24h` (retention과 일치) |
| auto.create.topics | true | 프로덕션 false (오타 토픽 방지, 파티션·RF 통제) | true (데모 — 자동 생성 토픽도 `num.partitions=3` 적용) |
| cleanup.policy | delete | delete 유지 (텔레메트리는 시간 경과로 가치 소멸) | delete |

### 프로듀서 (kafka exporter)

| 설정 | exporter 기본값 | 권장 | apm-infra |
|------|---------------|------|-----------|
| required_acks | 1 (리더 확인만) | 텔레메트리는 1이 통례 (지연 우선), 유실 불허면 -1 | 1 |
| compression | none | **zstd** — 텔레메트리는 반복 패턴이 많아 압축률 압도적 (벤치마크: zstd 23~26% vs lz4 40.7%로 축소) | zstd |
| linger | 10ms | 기본 유지~상향 (배칭·압축 효율) | 10ms |

### 컨슈머 (다음 단계 spring-kafka 앱의 사전 지식)

| 설정 | 기본값 | 적재 파이프라인 권장 | 이유 |
|------|--------|---------------------|------|
| auto.offset.reset | latest | **earliest** | latest는 신규 그룹이 기존 데이터를 건너뜀 → 유실 |
| enable.auto.commit | true | **false** + AckMode BATCH/MANUAL | "INSERT 성공 → 커밋" 순서로 at-least-once 보장. 크래시 시 유실 대신 중복 |
| max.poll.records | 500 | 1000~5000 | poll 1회 = ClickHouse bulk INSERT 1회. 단 `max.poll.interval.ms`(5분) 내 처리 |
| fetch.min.bytes / max.wait.ms | 1B / 500ms | 64KB+ / 500ms | 브로커가 모아서 응답 → 큰 배치 수신 |
| concurrency | 1 | 파티션 수 이하 (=3) | 초과분은 유휴. 과다하면 ClickHouse 파트 증가로 merge 부하 |
| 중복 흡수 | — | ClickHouse ReplacingMergeTree + insert 블록 dedup | at-least-once의 중복을 저장소에서 제거. ReplacingMergeTree의 dedup은 백그라운드 merge 시점(비동기) — 정확성 필요 조회는 FINAL |
| lag 모니터링 | — | `kafka_consumergroup_lag` 절대값 + 증가율 | "정적 lag 1,000은 무해, 초당 100씩 증가는 즉시 대응". 배치 적재는 톱니형 lag이 정상이라 `for:` 절로 오탐 방지 |

---

## 참고 자료

**국내**
- 카카오페이증권 — [일 41TB, 200억 건의 로그를 ClickStack으로 실시간 처리하기 (Pallas v2)](https://tech.kakaopay.com/post/pallas-v2-log-platform/)
- 카카오 — [전사 리소스 모니터링 시스템 KEMI](https://tech.kakao.com/2016/08/25/kemi/)
- LINE — [LINE에서 Kafka를 사용하는 방법 1편](https://engineering.linecorp.com/ko/blog/how-to-use-kafka-in-line-1/) / [2편](https://engineering.linecorp.com/ko/blog/how-to-use-kafka-in-line-2)
- 토스증권 — [MSA 환경 Observability 높이기](https://toss.tech/article/MSA-observability)
- 우아한형제들 — [ELK Stack에서 Loki로 전환한 이유](https://techblog.woowahan.com/14505/) (Kafka 미사용 반례)

**글로벌**
- Uber — [Fast and reliable schema-agnostic log analytics platform](https://www.uber.com/us/en/blog/logging/)
- Netflix Keystone — [InfoQ 정리](https://www.infoq.com/news/2016/03/netflix-keystone-data-pipeline/)
- LinkedIn — [Running Kafka At Scale](https://engineering.linkedin.com/kafka/running-kafka-scale)
- Cloudflare — [Cloudflare's logging pipeline](https://blog.cloudflare.com/an-overview-of-cloudflares-logging-pipeline/) / [Log analytics using ClickHouse](https://blog.cloudflare.com/log-analytics-using-clickhouse/)
- Datadog — [Introducing Husky](https://www.datadoghq.com/blog/engineering/introducing-husky/)
- Grafana Mimir — [About ingest storage architecture](https://grafana.com/docs/mimir/latest/get-started/about-grafana-mimir-architecture/about-ingest-storage-architecture/)

**공식 문서**
- ClickHouse — [Integrating OpenTelemetry](https://clickhouse.com/docs/observability/integrating-opentelemetry) ("필요할 때만 Kafka")
- OTel Collector — [kafkaexporter v0.156.0 README](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.156.0/exporter/kafkaexporter/README.md)
- OTLP proto — [trace_service.proto](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/collector/trace/v1/trace_service.proto)

**초기 설정값 (9절)**
- Confluent — [How to choose the number of topics/partitions](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/) / [Kafka message compression](https://www.confluent.io/blog/apache-kafka-message-compression/)
- New Relic — [Kafka best practices](https://newrelic.com/blog/observability/kafka-best-practices) / segment.ms-retention 불일치 사례: [Real-Time Event Processing at New Relic](https://blog.newrelic.com/engineering/apache-kafka-event-processing/)
- AWS MSK — [Default configuration](https://docs.aws.amazon.com/msk/latest/developerguide/msk-default-configuration.html)
- Kafka 공식 — [Producer configs](https://kafka.apache.org/41/configuration/producer-configs/) / [Consumer configs](https://kafka.apache.org/41/configuration/consumer-configs/) / [Topic configs](https://kafka.apache.org/41/configuration/topic-configs/)
- spring-kafka — [@KafkaListener 배치 리스너](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/listener-annotation.html)
- ClickHouse — [Deduplication strategies](https://clickhouse.com/docs/guides/developer/deduplication)
