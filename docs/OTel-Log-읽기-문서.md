# OTel Log 읽기 문서

> OTel Collector의 debug exporter가 출력하는 로그(트레이스/메트릭)를 처음 보는 사람을 위한 해설.
> 실제 apm-demo 주문 생성 요청의 로그를 예시로 사용한다.

---

## 1. 큰 그림 — 이 로그에는 3종류의 신호가 섞여 나온다

`docker logs -f apm-otel-collector`에 출력되는 내용은 전부 아래 3가지 중 하나다. 각 블록의 시작 줄에서 `"otelcol.signal"` 값으로 구분한다.

| 신호 | 시작 줄 예시 | 내용 |
|------|-------------|------|
| **traces** | `info Traces {..., "spans": 15}` | 요청의 실행 경로 (span 묶음) |
| **metrics** | `info Metrics {..., "metrics": 13}` | 수치 지표 (메모리, 응답시간 분포 등) |
| **logs** | `info Logs {...}` | 애플리케이션 로그 |

중요한 성질 하나: **로그는 요청 즉시 찍히지 않는다.** Agent와 Collector 양쪽에 배치(모아서 보내기) 단계가 있어서, API를 호출하고 수 초 뒤에 여러 건이 한 덩어리로 출력된다. 위 예시의 `"spans": 15`는 "span 15개가 한 배치로 도착했다"는 뜻이다.

---

## 2. 트레이스 블록의 구조 — 위에서 아래로 3층

```
ResourceSpans #0
├── Resource attributes        ← ① 누가 보냈나 (앱 단위 정보)
├── ScopeSpans #0
│   ├── InstrumentationScope   ← ② 어떤 계측기가 만들었나
│   └── Span #0, #1, ...       ← ③ 실제 데이터
├── ScopeSpans #1
│   └── ...
```

### ① Resource attributes — 발신자 명함

```
-> service.name: Str(apm-demo)          ← 제일 먼저 볼 것. 어느 서비스의 데이터인가
-> process.runtime.version: Str(21.0.10+7-LTS)
-> host.name: Str(kitttae.local)
-> telemetry.distro.version: Str(2.29.0)  ← Agent 버전
```

길고 반복돼서 시끄럽지만, 전부 "이 데이터를 보낸 프로세스가 누구인가"다. 여러 앱이 붙으면 `service.name`으로 구분한다.

### ② InstrumentationScope — 계측기 서명

```
InstrumentationScope io.opentelemetry.tomcat-10.0 2.29.0-alpha
```

Agent 안의 어떤 계측 모듈이 이 span을 만들었는지다. apm-demo에서 보게 되는 4종:

| 스코프 | 만드는 span |
|--------|------------|
| `io.opentelemetry.tomcat-10.0` | HTTP 요청 수신 (`POST /api/orders`) |
| `io.opentelemetry.spring-data-1.8` | 리포지토리 호출 (`OrderRepository.save`) |
| `io.opentelemetry.hibernate-6.0` | ORM 동작 (`Session.persist`, `Transaction.commit`) |
| `io.opentelemetry.jdbc` | 실제 SQL (`INSERT apm_demo.orders`) |

### ③ Span — 필드별 의미

```
Span #0
    Trace ID       : 6941cd3b238a16ff02a81ba7da828818   ← 요청 전체의 ID
    Parent ID      : 41a893475d71cad2                   ← 부모 span의 ID (비어 있으면 루트)
    ID             : 8638cb225b1abf54                   ← 이 span 자신의 ID
    Name           : OrderRepository.save               ← 무슨 동작인가
    Kind           : Internal                           ← 동작의 종류
    Start time / End time                               ← 소요시간 = End - Start
    Status code    : Unset                              ← 에러 여부 (아래 5절 참고)
Attributes:
    -> code.function: Str(save)                         ← 동작별 상세 정보
```

**Kind** 3종만 알면 된다:

- `Server` — 요청이 **들어온** 지점 (트레이스의 입구)
- `Client` — 외부로 **나간** 호출 (DB, 다른 서비스)
- `Internal` — 앱 내부 함수 호출

---

## 3. 읽기 규칙 2개 — 이것만 알면 트레이스가 조립된다

1. **같은 Trace ID = 같은 요청.** 로그에 span이 뒤섞여 나와도(스코프별로 그룹핑돼서 시간순이 아님), Trace ID가 같으면 전부 한 요청의 조각이다.
2. **Parent ID를 따라가면 나무가 된다.** Parent ID가 빈 span이 루트(요청 입구)고, 각 span의 Parent ID는 자기를 호출한 span의 ID를 가리킨다.

---

## 4. 시나리오 — 주문 생성 1건을 로그에서 재조립하기

Swagger에서 `POST /api/orders`를 1번 누르면, 로그 곳곳에 흩어진 span 5개가 나온다. Trace ID `6941cd3b...`로 모아서 Parent ID로 조립하면:

```
[루트] POST /api/orders            Server    41a893...  (Parent 없음, status_code=201)
  ├── OrderRepository.save         Internal  8638cb...  (Parent: 41a893...)
  │     └── Session.persist Order  Internal  bbb988...  (Parent: 8638cb...)
  │           └── INSERT orders    Client    925c27...  (Parent: bbb988...)
  └── Transaction.commit           Internal  9d1ba9...  (Parent: 41a893...)
```

이 나무에서 읽을 수 있는 것:

- **요청 전체 소요시간**: 루트 span의 End − Start = 774601 → 796049 ≈ **21ms**
- **그중 DB가 쓴 시간**: INSERT span ≈ 2.5ms → "느리면 어디가 느린가"를 층별로 쪼갤 수 있다
- **실행된 SQL 전문**: `db.statement: insert into orders (created_at,customer_id,...) values (?,?,...)`
- **응답 코드**: `http.response.status_code: Int(201)`
- **누가 호출했나**: `user_agent.original: ...Chrome...` (Swagger UI에서 호출한 것까지 보임)

이게 APM의 최소 단위 경험이다 — 나중에 이 로그가 ClickHouse에 저장되고 Grafana에서 나무 모양으로 그려지는 것뿐, 데이터 자체는 지금 보는 것과 동일하다.

---

## 5. 여기서 꼭 봐야 하는 체크포인트

| 확인 항목 | 보는 곳 | 정상 기준 |
|----------|--------|----------|
| 내 앱 데이터가 맞는가 | Resource의 `service.name` | `apm-demo` |
| 요청이 트레이스로 묶였는가 | 같은 Trace ID의 span 개수 | 주문 생성 기준 5개 |
| 계층이 연결됐는가 | Parent ID 체인 | 루트(Server) → Internal → Client |
| 요청이 성공했는가 | `http.response.status_code` | 201/200 |
| 에러 span 표시 | `Status code` 필드 | 정상=`Unset`, 에러=`Error` |

**헷갈리기 쉬운 것**: `Status code: Unset`은 "에러 아님"이라는 뜻이다 (Ok가 아니라 Unset이 정상값). 그리고 OTel 규약상 **Server span은 5xx만 Error로 표시**한다 — 409 같은 4xx는 `http.response.status_code` 속성으로만 남는다. 즉 "취소 불가(409)" 요청도 Status code는 Unset이다.

---

## 6. 메트릭 블록 읽기 (요약)

```
Metric #N
    name=http.server.request.duration      ← 지표 이름
    unit=s, type=HISTOGRAM                  ← 단위와 형태
    attributes={http.route=/api/orders, http.response.status_code=201}
    getCount=2, getSum=0.29                 ← 2번 호출, 합계 0.29초
    getBoundaries=[0.005, 0.01, ...]        ← 히스토그램 버킷 경계
    getCounts=[0, 0, 0, 1, ...]             ← 각 버킷에 떨어진 요청 수
```

자주 보게 될 이름들: `http.server.request.duration`(엔드포인트별 응답시간 분포 — p99의 원천), `jvm.memory.used`/`jvm.gc.duration`(JVM 상태), `db.client.connections.*`(HikariCP 커넥션 풀 — `pending_requests`가 0보다 크면 풀 고갈 신호).

트레이스가 "요청 하나의 이야기"라면, 메트릭은 "구간별 통계"다. 히스토그램 버킷 구조라 나중에 Grafana에서 p95/p99를 계산할 수 있다.

---

## 7. 이 데이터가 Kafka에 어떻게 들어가는가

다음 단계에서 Collector의 exporter를 `debug` → `kafka`로 바꾸면, **지금 눈으로 보고 있는 이 구조가 그대로 OTLP Protobuf 바이너리로 토픽에 들어간다.** 사람이 읽기 좋게 풀어 쓴 게 debug 출력이고, 기계용 직렬화가 Protobuf일 뿐 내용은 동일하다.

| 토픽 | 들어가는 것 | 위 로그에서의 대응 |
|------|------------|------------------|
| `traces` | ResourceSpans (Resource + ScopeSpans + Span 전체) | 2~4절에서 본 트레이스 블록 |
| `metrics` | ResourceMetrics | 6절에서 본 메트릭 블록 |

컨슈머(Spring Boot)가 할 일이 여기서 정해진다:

1. 토픽에서 Protobuf 바이너리를 꺼내 `opentelemetry-proto` 라이브러리로 역직렬화
2. 위 구조에서 저장할 필드를 뽑아 ClickHouse에 배치 INSERT

ClickHouse `traces` 테이블에 뽑아 넣을 필드 후보 — 전부 이 문서에서 이미 본 것들이다:

| 컬럼 후보 | 출처 |
|----------|------|
| trace_id, span_id, parent_span_id | Span의 Trace ID / ID / Parent ID |
| name, kind | Span의 Name / Kind |
| start_time, duration | Start time, End−Start 계산 |
| status_code | Span의 Status code |
| service_name | Resource의 `service.name` |
| http_route, http_status_code | Attributes (`http.route`, `http.response.status_code`) |
| db_statement, db_system | Attributes (`db.statement`, `db.system`) |

즉 **"debug 로그를 읽을 줄 안다 = 컨슈머가 파싱할 데이터 구조를 안다"**이다. 이 문서의 구조 이해가 그대로 컨슈머 구현의 사전 지식이 된다.
