# apm-infra

apm-kitae 인프라 실행 구성. Docker Compose로 로컬 개발/데모 환경의 인프라 컨테이너를 관리한다.

## 서비스 목록

| 서비스 | 이미지 | 포트 | 용도 |
|--------|--------|------|------|
| mysql | mysql:8.0 | 3306 | apm-demo 로컬 개발용 DB |
| otel-collector | otel/opentelemetry-collector-contrib:0.156.0 | 4317 (OTLP gRPC), 4318 (OTLP HTTP), 13133 (health) | 관측 데이터 수집 게이트웨이 |
| kafka | apache/kafka:4.0.0 | 9092 (EXTERNAL) | 텔레메트리 버퍼 (Collector → 컨슈머) |

> 이후 추가 예정: ClickHouse, MinIO, Grafana

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

> `MYSQL_USER`/`MYSQL_PASSWORD`/`MYSQL_DATABASE`는 볼륨이 비어 있을 때 최초 1회만 적용된다. 값 변경 시 `docker compose down -v` 후 재기동.

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

### 수신 검증

```bash
docker logs -f apm-otel-collector  # 터미널 1: 수신 로그 감시

# 터미널 2: apm-demo를 otlp exporter로 실행 후 API 호출
curl -X POST http://localhost:18080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"customerId":"customer-1","productId":"product-1","quantity":2,"unitPrice":4500}'
```

터미널 1에 `POST /api/orders` SERVER span과 JDBC CLIENT span이 같은 traceId로 출력되면 정상.
