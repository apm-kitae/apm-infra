# apm-infra

apm-kitae 인프라 실행 구성. Docker Compose로 로컬 개발/데모 환경의 인프라 컨테이너를 관리한다.

## 서비스 목록

| 서비스 | 이미지 | 포트 | 용도 |
|--------|--------|------|------|
| mysql | mysql:8.0 | 3306 | apm-demo 로컬 개발용 DB |
| otel-collector | otel/opentelemetry-collector-contrib:0.156.0 | 4317 (OTLP gRPC), 4318 (OTLP HTTP), 13133 (health) | 관측 데이터 수집 게이트웨이 |

> 이후 추가 예정: Kafka, ClickHouse, MinIO, Grafana

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

> `MYSQL_USER`/`MYSQL_PASSWORD`/`MYSQL_DATABASE`는 볼륨이 비어 있을 때 최초 1회만 적용된다. 값 변경 시 `docker compose down -v` 후 재기동.

## 접속 확인

```bash
mysql -h127.0.0.1 -P3306 -uapm -p1234 apm_demo
```

## OTel Collector

파이프라인 구성 (`otel-collector/config.yaml`):

```
receivers: otlp (gRPC :4317, HTTP :4318)  →  processors: batch  →  exporters: debug (verbosity: detailed)
파이프라인: traces / metrics / logs 3종 동일 구성 + health_check extension (:13133)
```

> OTel Java Agent의 OTLP 기본 프로토콜은 `http/protobuf`(4318)이므로 HTTP 리시버 필수. gRPC(4317)는 `-Dotel.exporter.otlp.protocol=grpc` 지정 시 사용.
> logs 파이프라인 포함 — Agent 기본값이 traces/metrics/logs 모두 export하므로, logs 파이프라인이 없으면 앱 콘솔에 export 실패 에러가 반복 출력된다.

- traces / metrics 파이프라인을 분리 정의 (이후 Kafka 토픽 분기 대비)
- `debug` exporter는 수신한 span/메트릭 상세를 컨테이너 로그에 출력 — 수신 검증 용도
- contrib 이미지 사용: kafka exporter 내장 (core 이미지에는 없음)

### 수신 검증

```bash
docker logs -f apm-otel-collector  # 터미널 1: 수신 로그 감시

# 터미널 2: apm-demo를 otlp exporter로 실행 후 API 호출
curl -X POST http://localhost:18080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"customerId":"customer-1","productId":"product-1","quantity":2,"unitPrice":4500}'
```

터미널 1에 `POST /api/orders` SERVER span과 JDBC CLIENT span이 같은 traceId로 출력되면 정상.
