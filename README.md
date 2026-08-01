# apm-infra

apm-kitae 인프라 실행 구성. Docker Compose로 로컬 개발/데모 환경의 인프라 컨테이너를 관리한다.

## 서비스 목록

| 서비스 | 이미지 | 포트 | 용도 |
|--------|--------|------|------|
| mysql | mysql:8.0 | 3306 | apm-demo 로컬 개발용 DB |
| otel-collector | otel/opentelemetry-collector-contrib:0.156.0 | 4317 (OTLP gRPC), 4318 (OTLP HTTP), 13133 (health) | 관측 데이터 수집 게이트웨이 |
| kafka | apache/kafka:4.0.0 | 9092 (EXTERNAL) | 텔레메트리 버퍼 (Collector → 컨슈머) |
| kafka-ui | ghcr.io/kafbat/kafka-ui:v1.5.0 | 8081 | 토픽·파티션·컨슈머 그룹 관찰 UI |
| clickhouse | clickhouse/clickhouse-server:25.3.14.14 | 8123 (HTTP), 9000 (native) | 텔레메트리 영속 저장소 |
| grafana | grafana/grafana-oss:12.4.3 | 3000 | ClickHouse 대시보드 (서비스 맵·서비스 상세·HTTP 성능·JVM·트레이스 검색) |

> 이후 추가 예정: MinIO

## 실행

```bash
cp .env.example .env      # 최초 1회, 필요 시 값 수정
docker compose up -d
docker compose ps                    # mysql은 (healthy) 표시
curl -s localhost:13133 | head -c 80 # collector 상태 확인 (health_check extension)
```

종료:

```bash
docker compose down       # 컨테이너만 제거 (데이터 유지)
docker compose down -v    # 볼륨까지 제거 (DB 초기화)
```

## 환경변수 (.env)

| 변수 | 기본값 | 설명 |
|------|--------|------|
| MYSQL_ROOT_PASSWORD | 1234 | root 비밀번호 |
| MYSQL_DATABASE | apm_demo | 초기 생성 DB |
| MYSQL_USER | apm | 애플리케이션 계정 |
| MYSQL_PASSWORD | 1234 | 애플리케이션 계정 비밀번호 |
| MYSQL_HOST | localhost | 앱에서 접속할 호스트 |
| MYSQL_PORT | 3306 | 호스트 포트 매핑 (로컬 mysqld가 3306 점유 시 13306 등으로 변경) |
| OTEL_GRPC_PORT | 4317 | Collector OTLP gRPC 호스트 포트 |
| OTEL_HTTP_PORT | 4318 | Collector OTLP HTTP 호스트 포트 |
| OTEL_HEALTH_PORT | 13133 | Collector health_check 호스트 포트 |
| KAFKA_PORT | 9092 | Kafka EXTERNAL 리스너 호스트 포트 (호스트의 컨슈머 앱·CLI 접속용) |
| KAFKA_UI_PORT | 8081 | kafka-ui 웹 접속 호스트 포트 (8080은 로컬 Spring Boot·타 컨테이너와 충돌 잦음) |
| CLICKHOUSE_HTTP_PORT | 8123 | ClickHouse HTTP 호스트 포트 (clickhouse-client HTTP·Grafana) |
| CLICKHOUSE_TCP_PORT | 9000 | ClickHouse native TCP 호스트 포트 (컨슈머 앱 드라이버) |
| CLICKHOUSE_DB | otel | 초기 생성 DB |
| CLICKHOUSE_USER | apm | 애플리케이션 계정 |
| CLICKHOUSE_PASSWORD | 1234 | 애플리케이션 계정 비밀번호 |
| GRAFANA_PORT | 3000 | Grafana 웹 접속 호스트 포트 (로컬 프론트 개발 서버와 충돌 시 13000 등으로 변경) |
| GRAFANA_USER | apm | Grafana admin 계정 |
| GRAFANA_PASSWORD | 1234 | Grafana admin 비밀번호 |

> `MYSQL_USER`/`MYSQL_PASSWORD`/`MYSQL_DATABASE`는 볼륨이 비어 있을 때 최초 1회만 적용된다. 값 변경 시 `docker compose down -v` 후 재기동.
> ClickHouse 초기화 SQL(`clickhouse/init/`)도 볼륨이 비어 있을 때만 실행된다. 스키마 변경 시 `docker compose down -v` 후 재기동.

## 접속 확인

```bash
mysql -h127.0.0.1 -P3306 -uapm -p1234 apm_demo
```

## OTel Collector

파이프라인 구성 (`otel-collector/config.yaml`):

```
receivers: otlp (gRPC :4317, HTTP :4318)  →  processors: batch  →  exporters:
  traces  → debug + kafka (topic: traces,  키=trace_id)
  metrics → debug + kafka (topic: metrics)
  logs    → debug (토픽 미전송)
+ health_check extension (:13133)
```

> OTel Java Agent의 OTLP 기본 프로토콜은 `http/protobuf`(4318)이므로 HTTP 리시버 필수. gRPC(4317)는 `-Dotel.exporter.otlp.protocol=grpc` 지정 시 사용.
> logs 파이프라인 포함 — Agent 기본값이 traces/metrics/logs 모두 export하므로, logs 파이프라인이 없으면 앱 콘솔에 export 실패 에러가 반복 출력된다.

- `debug` exporter는 수신한 span/메트릭 상세를 컨테이너 로그에 출력 — 수신 검증 용도
- `kafka` exporter는 동일 데이터를 OTLP Protobuf(`otlp_proto`)로 토픽에 적재 — 컨슈머 앱의 입력
- contrib 이미지 사용: kafka exporter 내장 (core 이미지에는 없음)

## Kafka

KRaft 단일 노드 (ZooKeeper 없음). 리스너 2개:

| 리스너 | 주소 | 접속 주체 |
|--------|------|----------|
| INTERNAL | `kafka:29092` | 같은 compose 네트워크의 컨테이너 (Collector) |
| EXTERNAL | `localhost:9092` | 호스트의 컨슈머 앱, CLI |

| 토픽 | 파티션 | 메시지 키 | 값 |
|------|--------|----------|-----|
| traces | 3 | trace_id (`partition_traces_by_id`) — 같은 트레이스의 span은 같은 파티션 | OTLP Protobuf (ExportTraceServiceRequest) |
| metrics | 3 | 없음 (라운드로빈) | OTLP Protobuf (ExportMetricsServiceRequest) |

- 토픽은 첫 메시지 수신 시 자동 생성 (`KAFKA_NUM_PARTITIONS: 3`)
- retention 24시간 — 컨슈머가 ClickHouse로 옮기는 버퍼 용도라 짧게 유지

### 적재 검증

```bash
# 토픽 목록·파티션 확인
docker exec apm-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic traces

# 메시지 소비 (키=trace_id 확인, 값은 Protobuf 바이너리)
docker exec apm-kafka /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic traces --from-beginning \
  --property print.key=true --property print.partition=true
```

apm-demo API 호출 후 위 컨슈머에 `Partition:N  <trace_id>  <바이너리>` 형식으로 출력되면 정상.

### kafka-ui

`http://localhost:8081` 접속 — 클러스터명 `apm-local`, `kafka:29092` INTERNAL 리스너로 연결.

- **Topics**: `traces` / `metrics` 토픽, 파티션 수(3), 파티션별 오프셋
- **Messages**: 토픽 선택 → Messages 탭 — 키(trace_id 16진수)와 파티션 번호 확인. 값은 ProtobufFile serde가 OTLP 스키마로 JSON 디코딩해 표시 (Protobuf 바이너리 원문 해설은 [Kafka 적재 데이터 읽기 문서](./docs/Kafka-적재-데이터-읽기-문서.md))
- **Consumers**: 컨슈머 그룹별 lag — 컨슈머 앱(Spring Boot) 개발 시 파티션 분배·처리 지연 관찰용

value 디코딩 (ProtobufFile serde):

| 토픽 | 매핑 메시지 타입 |
|------|------------------|
| traces | `opentelemetry.proto.collector.trace.v1.ExportTraceServiceRequest` |
| metrics | `opentelemetry.proto.collector.metrics.v1.ExportMetricsServiceRequest` |

- proto 스키마: `kafka-ui/proto/` (opentelemetry-proto v1.10.0, 컨테이너에 `/protofiles`로 마운트)
- 디코딩된 JSON의 `traceId`/`spanId`는 base64 표시 — 메시지 키의 16진수와 같은 값의 다른 인코딩

### 수신 검증

```bash
docker logs -f apm-otel-collector  # 터미널 1: 수신 로그 감시

# 터미널 2: apm-demo를 otlp exporter로 실행 후 API 호출
curl -X POST http://localhost:18080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"customerId":"customer-1","productId":"product-1","quantity":2,"unitPrice":4500}'
```

터미널 1에 `POST /api/orders` SERVER span과 JDBC CLIENT span이 같은 traceId로 출력되면 정상.

## ClickHouse

텔레메트리 영속 저장소. OTel ClickHouse exporter 스키마를 준용한 wide denormalized table. 컨슈머 앱(Spring Boot, 별도 레포)이 Kafka에서 소비한 OTLP를 역직렬화해 INSERT하는 대상 — 이 레포는 컨테이너와 스키마까지만 담당한다.

| 테이블 | 단위 | 용도 |
|--------|------|------|
| `otel_traces` | span 1개 = 1행 | 트레이스. `ParentSpanId → SpanId`로 트리 재구성, `TraceId` bloom filter로 단건 조회 |
| `otel_metrics_gauge` | 데이터포인트 | 순간값 (jvm.cpu.recent_utilization) |
| `otel_metrics_sum` | 데이터포인트 | 카운터·UpDownCounter (jvm.cpu.time, jvm.memory.used, jvm.thread.count) |
| `otel_metrics_histogram` | 데이터포인트 | 분포 (http.server.request.duration, jvm.gc.duration) |

- 스키마 초기화: `clickhouse/init/*.sql` (볼륨이 빈 첫 기동 시 파일명 순 실행)
- 전 테이블 MergeTree, `PARTITION BY toDate(...)`, TTL 30일
- 설계 근거(wide table 채택, metrics 타입 분리, Exemplar로 metrics→traces 연결): [ClickHouse 스키마 문서](./docs/ClickHouse-스키마-문서.md)

### 접속 확인

```bash
# clickhouse-client (컨테이너 내부)
docker exec -it apm-clickhouse clickhouse-client -u apm --password 1234

# 테이블 목록
docker exec apm-clickhouse clickhouse-client -u apm --password 1234 --query "SHOW TABLES FROM otel"

# HTTP 인터페이스 (호스트)
curl -s "http://localhost:8123/?user=apm&password=1234" --data-binary "SELECT count() FROM otel.otel_traces"
```

## Grafana

ClickHouse에 적재된 traces/metrics 대시보드. datasource와 대시보드 전부 provisioning 파일로 관리 — `docker compose up`만으로 수동 설정 없이 동작한다.

- 접속: http://localhost:3000 (계정 `apm`/`1234`, `.env`로 변경 가능)
- datasource: `grafana/provisioning/datasources/clickhouse.yml` — 컨테이너 네트워크 `clickhouse:8123` 연결, 계정은 compose 환경변수(`CLICKHOUSE_*`) 보간
- 대시보드: `grafana/dashboards/*.json` — APM 폴더로 자동 로드. UI 편집 불가(파일이 원본), 수정은 JSON 편집 후 `docker compose restart grafana`

| 대시보드 | 근거 데이터 | 내용 |
|----------|------------|------|
| APM / 서비스 맵 | `otel_traces` (span) | 서비스 간 호출 그래프(Node Graph). 노드 = 받은 요청·평균 지연·에러 비율, 엣지 = 서비스 경계를 넘은 호출. 서비스명 클릭 → 서비스 상세 |
| APM / 서비스 상세 | `otel_traces` + `otel_metrics_*` | `$service` 선택. 요청 수·에러율·p95, 분당 요청·에러, 엔드포인트별 지표, 느린 요청(TraceId 클릭 → 워터폴), 힙·스레드·CPU |
| APM / HTTP 성능 | `otel_metrics_histogram` | 분당 요청 수·평균 응답 시간(서비스·라우트별), 기간 p50/p95/p99 (버킷 상한 근사) |
| APM / JVM | `otel_metrics_sum`·`gauge`·`histogram` | 힙 메모리, 스레드 수, CPU 사용률, 분당 GC 횟수·평균 GC 소요, 로드된 클래스 수 (전부 서비스별) |
| APM / 트레이스 검색 | `otel_traces` | 응답 시간 산점도 → 느린 트레이스 목록에서 **TraceId 클릭** → 워터폴(span 계층·구간 막대) + span 상세 + span event |

**메트릭 기반 대시보드**(HTTP 성능·JVM)는 OTel Java Agent 기본값인 **cumulative** temporality로 적재되므로, 요청 수·GC 횟수 같은 카운터 패널은 서비스·시리즈별 인접 구간 차분(window `lagInFrame`)으로, 기간 백분위는 창 양끝 `BucketCounts` 차분으로 계산한다. 데이터가 비어 있으면 apm-consumer가 metrics 토픽을 소비 중인지 먼저 확인.

**span 기반 대시보드**(서비스 맵·서비스 상세)는 차분 없이 span을 직접 집계한다. 같은 엔드포인트라도 p95가 HTTP 성능 대시보드와 다르게 나오는데, 그쪽은 히스토그램 버킷 상한 근사이고 이쪽은 실측 분포라 **정상이다**. 헬스체크·Swagger 요청(`GET /actuator%`, `GET /swagger%`, `GET /v3/api-docs%`)은 제외한다 — 빼지 않으면 요청 수의 3분의 1이 헬스체크가 된다.

> span 기반 집계는 **전량 저장**을 전제로 한다. 현재 샘플러를 지정하지 않아 Agent 기본값 `parentbased_always_on`이 적용된다. 나중에 tail 샘플링을 도입하면 요청 수가 실제보다 작아지므로 메트릭 기반으로 옮겨야 한다.

### 서비스 맵 읽는 법

| 보이는 것 | 뜻 |
|-----------|-----|
| 노드가 하나만 | 상대 서비스에 OTel Agent가 붙지 않았거나, 그 서비스가 이 시간 범위에서 요청을 받지 않았다 |
| 노드는 둘인데 엣지가 없음 | 서비스 간 호출이 없었거나 `traceparent`가 전파되지 않았다 |
| 노드 테두리에 빨강 비율 | 그 서비스가 받은 요청의 에러 비율 |

계측 누락을 확인할 때는 시간 범위를 `now-2m` 정도로 좁힌다. TTL이 30일이라 창이 넓으면 과거 span 때문에 노드가 계속 보인다.

### 확인

```bash
curl -s http://localhost:3000/api/health                                        # 기동 확인
curl -s -u apm:1234 http://localhost:3000/api/datasources/uid/clickhouse-otel/health  # datasource 연결
```
