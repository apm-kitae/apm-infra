# APM UI 기획서

ClickHouse `otel.otel_traces` / `otel.otel_metrics_*`를 조회하는 읽기 전용 API(`apm-api`)와 React 화면(`apm-web`).
분산 트레이싱 — 하나의 TraceId에 이어진 apm-demo·apm-payment span을 워터폴로 펼치고, 어느 span이 시간을 썼는지 계산해 문장으로 보여준다.

---

## 1. 목적

Grafana 대시보드 5종은 PromQL/SQL과 패널 구조를 아는 사람이 읽는다. 이 UI는 그 지식 없이 아래 4개 질문에 답하는 것만 한다.

| 질문 | 화면 |
|------|------|
| **지금 정상인가, 아니면 뭐가 문제인가** | **개요 (판정·진단)** |
| 서비스가 몇 개고 서로 어떻게 부르나, 지금 어디가 아픈가 | 서비스 맵 |
| 이 서비스의 요청 수·에러율·p95는, 어느 엔드포인트가 느린가 | 서비스 상세 |
| 느린 요청 하나를 어떻게 찾나 | 트레이스 검색 |
| **그 요청의 1.8초는 어디서 쓰였나** | **트레이스 상세 (워터폴)** |

읽기 순서는 위에서 아래다. 판정 → 어느 서비스 → 어느 엔드포인트 → 어느 span. 각 단계가 다음 단계로 링크된다.

Grafana는 남긴다. 임의 SQL·라벨 드릴다운은 Grafana가 낫다. 이 UI가 존재할 이유는 굵게 표시한 두 화면이다.

- **개요** — 지표를 나열하는 대신 규칙으로 판정한다. 그래프 20개를 눈으로 훑어 이상을 찾는 방식을 대체한다
- **트레이스 상세** — span 트리에서 self time을 계산해 병목 span을 지목한다. Grafana Traces 패널은 self time을 계산하지 않아 부모 막대가 자식 시간을 포함한 채로 보인다

**span 기반이라 가능한 판정이 있다.** 지표만 있으면 "p99가 올랐다"까지가 전부지만, 부모-자식 관계가 있으면 "올라간 시간이 이 서비스 내부인지 하위 서비스 호출 대기인지"가 계산된다(6.7절 `DOWNSTREAM_LATENCY`). 이게 지표 기반 대시보드와 갈리는 지점이다.

---

## 2. 배치

| 레포 | 내용 | 포트 |
|------|------|------|
| `apm-api` (신규) | Spring Boot 3.3 / Java 21. ClickHouse 읽기 전용 조회 API | 18091 |
| `apm-web` (신규) | Vite + React 19 + TypeScript | 5173 |

`apm-consumer`에 조회 API를 얹지 않는 이유: 컨슈머는 `app.traces.buffer.max-rows: 10000` backpressure와 `spring.task.scheduling.pool.size: 2`(한쪽 flush가 다른 쪽 오프셋 커밋을 막지 않게)로 적재 경로를 지키고 있다. 같은 JVM에 조회를 얹으면 30일치 span에 대한 `quantileExact` 한 방이 ClickHouse 커넥션 풀과 톰캣 스레드를 함께 점유하고, 그 결과가 flush 지연 → 오프셋 커밋 지연 → Kafka lag 증가로 이어진다.

포트는 18080(demo) · 18081(payment) · 18090(consumer) · 8081(kafka-ui) · 3000(grafana) · 8123(clickhouse)과 겹치지 않게 18091 / 5173.

---

## 3. 데이터 흐름

```
apm-demo ─HTTP+traceparent─▶ apm-payment
   └─ OTel Agent ─▶ Collector ─▶ Kafka ─▶ apm-consumer ─▶ ClickHouse
                                                              │
                                                    ┌─────────┴─────────┐
                                                 Grafana            apm-api ─▶ apm-web
```

apm-api는 ClickHouse만 읽는다. Kafka·Collector와 직접 통신하지 않는다.

**적재 지연**: 컨슈머 flush 주기가 5초라 방금 만든 트레이스는 최대 5초 뒤 조회된다. 트레이스 상세에서 span 수가 예상보다 적으면 재조회 버튼으로 다시 부른다.

---

## 4. 화면 구성

### 4.1 서비스 맵 — `/`

SVG 노드 그래프. 노드 = 서비스, 엣지 = 서비스 경계를 넘은 호출.

```
   ┌────────────────┐            ┌────────────────┐
   │   apm-demo     │            │  apm-payment   │
   │  1,204 req     │──1,204──▶  │   1,204 req    │
   │  p95 340ms     │   82ms     │   p95 78ms     │
   │  ● 에러 0.4%   │            │  ● 에러 5.1%   │
   └────────────────┘            └────────────────┘
```

- 노드 테두리 링: 에러 비율만큼 빨강, 나머지 초록
- 에러율 ≥ 1%면 노드에 경고 표시
- 노드 클릭 → 서비스 상세
- 레이아웃은 직접 계산한다. 진입 엣지가 없는 노드를 레벨 0으로 두고 BFS로 레벨을 매겨 x = 레벨, y = 레벨 내 순번. 그래프 라이브러리를 넣지 않는다
  - `// ponytail: BFS 레이어 배치, 서비스 15개 넘어가면 힘 기반 레이아웃으로 교체`

### 4.2 서비스 상세 — `/services/:name`

- 상단: 요청 수 / 에러율 / p50 / p95 / p99 (선택 구간)
- 분당 요청·에러 라인 차트
- 엔드포인트 표: SpanName, 요청 수, p95, 에러율 — p95 내림차순
- 느린 요청 50건: 클릭 → 트레이스 상세
- JVM: 힙 사용량, 스레드 수, CPU 사용률

### 4.3 트레이스 검색 — `/traces`

- 필터: 서비스, 엔드포인트(SpanName), 최소 소요 시간(ms), 상태(전체/에러만), 시간 구간
- 산점도: x = 시각, y = 소요 시간(ms), 빨간 점 = 에러. 점 클릭 → 트레이스 상세
- 아래 목록: 느린 순 50건

### 4.4 트레이스 상세 — `/traces/:traceId` (핵심)

```
TraceId 4bf92f...  ·  총 1,842ms  ·  span 9개  ·  서비스 2개  ·  Error

┌─ 요약 ────────────────────────────────────────────────────────┐
│ ⚠ 시간의 82%를 apm-payment / POST /api/payments 가 자기 처리에 │
│   썼다 (self 1,510ms / 총 1,842ms)                             │
│ ⚠ apm-payment / POST /api/payments 에서 예외 발생              │
│   PaymentFailedException: injected failure                     │
│ ⚠ SELECT orders 가 같은 부모 아래 12회 반복 (합계 240ms)       │
└───────────────────────────────────────────────────────────────┘

                    0ms        500       1000      1500     1842
apm-demo    POST /api/orders   ████████████████████████████████  1842ms
apm-demo     └ INSERT orders   ██                                   18ms
apm-demo     └ POST /api/pay…    ███████████████████████████      1620ms
apm-payment    └ POST /api/pa…   ██████████████████████████       1580ms  ✕
apm-payment      └ INSERT pay…                    ███               70ms
apm-demo     └ UPDATE orders                                  █     12ms
```

- 막대의 **진한 부분 = self time**, 옅은 부분 = 자식이 쓴 시간. 부모 막대가 통째로 길어 보이는 착시를 없앤다
- 서비스별 색 구분. 서비스 경계를 넘는 지점에 구분선
- `StatusCode = Error`인 span은 자동 펼침 + `exception` event를 막대 아래 인라인 표시
- span 클릭 → 우측 패널에 SpanAttributes 전체, Events 목록(`order.status.changed` 등)
- 상단에 서비스 필터 칩 — apm-payment만 보기 등

---

## 5. API 명세

Base: `http://localhost:18091/api/v1`
시간 파라미터 `from`/`to`는 epoch millis. 전 엔드포인트 필수이며, 서버가 최대 24시간으로 제한한다(ClickHouse TTL은 30일이지만 한 번에 24시간 넘게 스캔할 이유가 없다).

공통 노이즈 필터 — 아래 조건은 모든 span 집계 쿼리에 들어간다. Java 상수 `SpanSql.NOISE_FILTER` 하나로 둔다.

```sql
SpanName NOT LIKE 'GET /actuator%'
AND SpanName NOT LIKE 'GET /swagger%'
AND SpanName NOT LIKE 'GET /v3/api-docs%'
```

빼지 않으면 요청 수의 3분의 1이 헬스체크가 된다.

### 5.1 `GET /services?from&to`

서비스 목록 + RED 지표.

```json
[
  { "name": "apm-demo", "requests": 1204, "errors": 5, "errorRate": 0.0042,
    "p50Ms": 42.1, "p95Ms": 340.5, "p99Ms": 1820.0 }
]
```

```sql
SELECT ServiceName, count() AS requests,
       countIf(StatusCode = 'Error') AS errors,
       quantileExact(0.50)(Duration) / 1e6 AS p50_ms,
       quantileExact(0.95)(Duration) / 1e6 AS p95_ms,
       quantileExact(0.99)(Duration) / 1e6 AS p99_ms
FROM otel.otel_traces
WHERE SpanKind = 'Server' AND Timestamp >= ? AND Timestamp <= ? AND {NOISE_FILTER}
GROUP BY ServiceName ORDER BY requests DESC
```

### 5.2 `GET /service-map?from&to`

```json
{
  "nodes": [ { "name": "apm-demo", "requests": 1204, "errorRate": 0.0042, "avgMs": 82.1 } ],
  "edges": [ { "source": "apm-demo", "target": "apm-payment", "calls": 1204, "avgMs": 82.1 } ]
}
```

노드 SQL은 5.1과 같은 형태. 엣지는 자기 조인 — 자식 SERVER span의 부모(= 호출한 서비스의 CLIENT span)를 찾아 서비스명이 다른 쌍만 센다.

```sql
SELECT p.ServiceName AS source, c.ServiceName AS target,
       count() AS calls, avg(c.Duration) / 1e6 AS avg_ms
FROM otel.otel_traces AS c
INNER JOIN otel.otel_traces AS p
        ON c.TraceId = p.TraceId AND c.ParentSpanId = p.SpanId
WHERE c.SpanKind = 'Server' AND c.ServiceName != p.ServiceName
  AND c.Timestamp >= ? AND c.Timestamp <= ? AND {NOISE_FILTER on c}
  AND p.Timestamp >= ? - INTERVAL 1 MINUTE AND p.Timestamp <= ?
GROUP BY source, target
```

부모 쪽 시간 하한을 1분 앞으로 미는 이유: 부모 span의 Timestamp는 자식보다 이르다. 구간 시작에 걸친 트레이스에서 부모만 범위 밖으로 떨어지면 엣지가 사라진다.

### 5.3 `GET /services/{name}/overview?from&to`

```json
{
  "name": "apm-demo",
  "requests": 1204, "errorRate": 0.0042, "p50Ms": 42.1, "p95Ms": 340.5, "p99Ms": 1820.0,
  "timeseries": [ { "t": 1753900800000, "requests": 21, "errors": 0 } ],
  "endpoints": [ { "spanName": "POST /api/orders", "requests": 402, "p95Ms": 1620.0, "errorRate": 0.012 } ]
}
```

시계열은 `toStartOfInterval(Timestamp, INTERVAL 1 minute)` 기준 분당 집계. 구간이 6시간을 넘으면 서버가 5분 단위로 바꾼다.

### 5.4 `GET /services/{name}/jvm?from&to`

```json
{
  "heapUsed": [ { "t": 1753900800000, "value": 268435456 } ],
  "threads":  [ { "t": 1753900800000, "value": 42 } ],
  "cpu":      [ { "t": 1753900800000, "value": 0.31 } ]
}
```

`jvm.memory.used`·`jvm.thread.count`는 UpDownCounter라 `otel_metrics_sum`, `jvm.cpu.recent_utilization`은 `otel_metrics_gauge`에 있다. sum 쪽은 시리즈(= `Attributes` 조합)가 여러 개라 **분 단위로 시리즈별 마지막 값을 고른 뒤 합산**해야 한다. 바로 `sum(Value)`를 하면 같은 분에 여러 번 export된 값이 중복 합산된다.

```sql
SELECT t, sum(v) AS value FROM (
  SELECT toStartOfInterval(TimeUnix, INTERVAL 1 minute) AS t,
         toString(Attributes) AS series,
         argMax(Value, TimeUnix) AS v
  FROM otel.otel_metrics_sum
  WHERE MetricName = 'jvm.memory.used' AND Attributes['jvm.memory.type'] = 'heap'
    AND ServiceName = ? AND TimeUnix >= ? AND TimeUnix <= ?
  GROUP BY t, series
) GROUP BY t ORDER BY t
```

### 5.5 `GET /traces?from&to&service&spanName&minDurationMs&status&limit`

산점도용 점과 느린 순 목록을 한 번에 준다. 화면 하나가 요청 하나를 쓴다.

| 파라미터 | 필수 | 기본값 |
|----------|------|--------|
| `from` / `to` | O | — |
| `service` | X | 전체 |
| `spanName` | X | 전체 |
| `minDurationMs` | X | 0 |
| `status` | X | `all` (`all` \| `error`) |
| `limit` | X | 50 (최대 200) |

```json
{
  "points": [ { "traceId": "4bf92f…", "t": 1753900812345, "durationMs": 1842.0, "error": true } ],
  "pointsTruncated": false,
  "items":  [ { "traceId": "4bf92f…", "t": 1753900812345, "service": "apm-demo",
                "spanName": "POST /api/orders", "durationMs": 1842.0, "statusCode": "Error" } ]
}
```

두 쿼리 모두 루트 span(`ParentSpanId = ''`)만 센다. 한 트레이스 = 한 점.

```sql
-- points
SELECT TraceId, Timestamp, Duration / 1e6 AS ms, StatusCode
FROM otel.otel_traces
WHERE ParentSpanId = '' AND Timestamp >= ? AND Timestamp <= ? AND {NOISE_FILTER}
  {AND ServiceName = ?} {AND SpanName = ?}
  {AND Duration >= ?} {AND StatusCode = 'Error'}
ORDER BY Timestamp LIMIT 2001

-- items
… 같은 WHERE … ORDER BY Duration DESC LIMIT ?
```

`LIMIT 2001`로 뽑아 2001건이면 마지막 1건을 버리고 `pointsTruncated: true`를 준다. 화면은 "구간에 트레이스가 많아 일부만 표시한다. 구간을 좁혀라"를 띄운다.

**조건절은 문자열 연결이 아니라 바인딩 파라미터로 만든다.** 조건 유무에 따라 SQL 조각과 파라미터 리스트를 함께 쌓는다. `service`·`spanName`은 사용자 입력이 그대로 들어오는 자리다.

### 5.6 `GET /traces/{traceId}` (핵심)

워터폴 한 화면에 필요한 전부. 응답 하나로 끝난다.

```json
{
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "rootSpanId": "00f067aa0ba902b7",
  "startedAt": 1753900812345,
  "durationMs": 1842.0,
  "spanCount": 9,
  "services": ["apm-demo", "apm-payment"],
  "hasError": true,
  "clockSkewDetected": false,
  "summary": [
    { "type": "BOTTLENECK", "message": "시간의 82%를 apm-payment / POST /api/payments 가 자기 처리에 썼다 (self 1,510ms / 총 1,842ms)", "spanId": "b7ad6b71…" },
    { "type": "ERROR", "message": "apm-payment / POST /api/payments 에서 PaymentFailedException: injected failure", "spanId": "b7ad6b71…" },
    { "type": "REPEATED_CALL", "message": "SELECT orders 가 같은 부모 아래 12회 반복 (합계 240ms)", "spanId": "c1d2e3f4…" }
  ],
  "spans": [
    {
      "spanId": "00f067aa0ba902b7",
      "parentSpanId": null,
      "depth": 0,
      "service": "apm-demo",
      "name": "POST /api/orders",
      "kind": "Server",
      "statusCode": "Error",
      "statusMessage": "",
      "startOffsetMs": 0.0,
      "durationMs": 1842.0,
      "selfTimeMs": 132.0,
      "attributes": { "order.id": "1", "order.status": "FAILED", "http.request.method": "POST" },
      "events": [
        { "t": 1753900812350, "name": "order.status.changed",
          "attributes": { "order.id": "1", "order.status": "PENDING" } }
      ]
    }
  ]
}
```

`spans`는 **트리 선주문 순서**로 정렬해 내려준다. 프론트는 `depth`만큼 들여쓰기하면 되고 트리를 다시 만들지 않는다.

```sql
SELECT Timestamp, SpanId, ParentSpanId, SpanName, SpanKind, ServiceName,
       Duration, StatusCode, StatusMessage, SpanAttributes,
       Events.Timestamp, Events.Name, Events.Attributes
FROM otel.otel_traces
WHERE TraceId = ?
ORDER BY Timestamp
LIMIT 2000
```

`TraceId`는 `ORDER BY` 키에 없지만 `idx_trace_id` bloom filter가 있어 단건 조회는 이 인덱스를 탄다.

없는 TraceId면 404.

---

## 6. 서버 계산 로직

SQL이 아니라 Java에서 계산한다. 여기가 이 API가 Grafana와 다른 지점이라 테스트를 붙인다.

### 6.1 트리 구성

1. `SpanId → Span` 맵을 만든다
2. `parentSpanId`가 비어 있거나 **맵에 없는** span을 루트로 본다
   - 맵에 없는 경우: 부모 span이 아직 적재되지 않았거나 다른 파티션에서 늦게 도착했다. 이때도 화면은 그려져야 한다
3. 루트가 여러 개면 시작 시각이 가장 이른 것을 `rootSpanId`로 삼고, 나머지는 그 뒤에 같은 depth로 붙인다
4. 각 노드의 자식을 시작 시각 오름차순 정렬 → 선주문 순회로 `spans` 배열과 `depth` 생성

### 6.2 self time

```
selfTime(S) = max(0, S.duration − 자식 구간의 합집합 길이)
```

자식은 **직계 자식만**, 그리고 각 자식 구간을 부모 구간으로 잘라낸 뒤 계산한다.

```java
/** 직계 자식 구간을 부모 구간으로 잘라 합집합 길이를 구하고, 부모 duration에서 뺀다. */
long selfTime(Span parent, List<Span> children) {
    long covered = 0;
    long cursor = parent.start();               // 여기까지는 이미 자식이 덮었다

    List<Span> sorted = children.stream()
            .sorted(comparingLong(Span::start)) // 시작 시각 오름차순
            .toList();

    for (Span c : sorted) {
        long cs = Math.max(c.start(), parent.start());   // 부모 구간으로 clamp
        long ce = Math.min(c.end(),   parent.end());
        if (ce <= cs) continue;                          // 부모 밖 (시계 오차)
        if (ce <= cursor) continue;                      // 앞 자식에 완전히 포함
        covered += ce - Math.max(cs, cursor);            // 겹친 부분은 한 번만
        cursor = ce;
    }
    return Math.max(0, parent.duration() - covered);
}
```

합집합으로 계산하는 이유: 자식이 동시 실행되면 단순 합이 부모 duration을 넘어 self time이 음수가 된다.

clamp가 필요한 이유: 자식 SERVER span의 타임스탬프는 다른 JVM 시계에서 온다. apm-payment 시계가 앞서면 자식 시작이 부모 시작보다 이르게 기록된다.

### 6.3 시계 오차 처리

`child.start < parent.start`인 span이 하나라도 있으면 `clockSkewDetected: true`를 세우고, 워터폴의 `startOffsetMs`는 `max(0, child.start − root.start)`로 눌러 음수 막대를 막는다. 화면은 배지로 "서비스 간 시계 오차가 있어 구간 위치가 정확하지 않다"를 표시한다. 시계 보정은 하지 않는다.

### 6.4 요약 규칙 3가지

| type | 조건 | 문장 |
|------|------|------|
| `BOTTLENECK` | self time 최대 span의 `selfTime / rootDuration ≥ 0.3` | `시간의 {n}%를 {service} / {span} 이 자기 처리에 썼다 (self {a}ms / 총 {b}ms)` |
| `ERROR` | `statusCode = Error`인 span 중 시작이 가장 이른 것 | `{service} / {span} 에서 {exception.type}: {exception.message}` — `exception` event가 없으면 `statusMessage` |
| `REPEATED_CALL` | 같은 `parentSpanId` + 같은 `name`인 span이 **10개 이상** | `{span} 이 같은 부모 아래 {n}회 반복 (합계 {ms}ms)` |

조건에 안 걸리면 그 항목은 배열에서 빠진다. 셋 다 안 걸리면 `summary: []`이고 화면은 요약 카드를 그리지 않는다.

`ERROR`에서 가장 이른 에러 span을 고르는 이유: 에러는 호출 스택을 타고 위로 전파돼 부모까지 전부 Error가 된다. 실제로 터진 곳은 가장 깊고 가장 이른 쪽이다.

---

## 7. apm-api 구성

```
com.apmkitae.api
├─ global/config/  CorsConfig, ClickHouseConfig, SwaggerConfig, GlobalExceptionHandler
├─ trace/          TraceController, TraceService, TraceRepository,
│                  SpanTreeBuilder, SelfTimeCalculator, TraceSummarizer, dto/
├─ service/        ServiceController, ServiceService, ServiceRepository, dto/
└─ common/         TimeRange(검증·최대 24h), SpanSql(NOISE_FILTER 등)
```

- `spring-boot-starter-jdbc` + `com.clickhouse:clickhouse-jdbc:0.9.0` — apm-consumer와 같은 드라이버. JPA 없음, `JdbcTemplate`만 쓴다
- `springdoc-openapi` — `/swagger-ui/index.html`
- CORS: `http://localhost:5173` 허용
- ClickHouse 쿼리 옵션에 `max_execution_time = 10`, `max_result_rows` 지정. 무거운 쿼리가 톰캣 스레드를 붙잡지 않게 한다
- 읽기 전용 계정으로 접속하는 편이 맞지만 현재 ClickHouse에는 `apm` 계정 하나뿐이다 — 계정 분리는 apm-infra 후속 과제

## 8. apm-web 구성

```
src/
├─ api/client.ts        axios 인스턴스 (baseURL = /api/v1, Vite proxy로 18091)
├─ api/types.ts         응답 DTO 타입
├─ pages/               ServiceMapPage · ServiceDetailPage · TraceSearchPage · TraceDetailPage
├─ components/          ServiceMap · Waterfall · SpanDetailPanel · TraceSummaryCard
│                       · LatencyScatter · TimeRangePicker
└─ lib/layout.ts        서비스 맵 BFS 레이어 배치
```

| 항목 | 선택 | 이유 |
|------|------|------|
| 빌드 | Vite | 신규 프로젝트. CRA는 유지보수가 끝났다 |
| 언어 | TypeScript | span 트리·self time·DTO 형태를 컴파일 시점에 잡는다 |
| 라우팅 | react-router-dom 7 | |
| 서버 상태 | @tanstack/react-query 5 | 시간 구간이 키. 재조회·로딩·에러 처리 |
| 차트 | recharts 3 | 라인·산점도만. React 19 peer 지원은 3.x부터 |
| 스타일 | Tailwind 4 | |

**워터폴과 서비스 맵에는 라이브러리를 쓰지 않는다.** 워터폴은 `depth` 들여쓰기 + 퍼센트 폭 div, 서비스 맵은 SVG `<rect>`/`<line>`이다. 그래프 레이아웃 라이브러리(react-flow 등)를 넣을 만한 규모가 아니다.

Vite proxy로 `/api` → `http://localhost:18091`.

---

## 9. 안전장치

| 항목 | 처리 |
|------|------|
| SQL 인젝션 | 전 파라미터 `?` 바인딩. 동적 WHERE는 SQL 조각과 파라미터를 함께 쌓는다 |
| 스캔 폭주 | `from`/`to` 필수, 최대 24시간. 초과 시 400 |
| 쿼리 시간 | ClickHouse `max_execution_time = 10` |
| 결과 폭주 | `points` 2000, `spans` 2000, `items` 200 상한 + `pointsTruncated` 플래그 |
| 루트 없는 트레이스 | 부모가 결과에 없는 span도 루트로 취급해 렌더 |
| 시계 오차 | 음수 오프셋 clamp + `clockSkewDetected` 배지 |
| 없는 TraceId | 404 + 화면에 "적재 지연일 수 있다, 5초 뒤 재조회" 안내 |
| 인증 | **없다.** 트레이스에 `order.id`·`customerId`가 들어 있어 공개 배포하면 노출된다. 로컬 실행 전제 — 배포는 범위 밖 |

---

## 10. TDD 계획

### apm-api (JUnit 5)

| 대상 | 검증 |
|------|------|
| `SelfTimeCalculator` | 자식 없음 → duration 그대로 / 자식 1개 포함 / 자식 2개 겹침(합집합) / 자식 2개 분리 / 자식이 부모 밖(clamp) / 자식 합이 부모 초과 → 0 하한 |
| `SpanTreeBuilder` | `ParentSpanId=''` 루트 / 부모 결측 span도 루트 / 루트 다중 시 가장 이른 것 선택 / 선주문 순서·depth / 형제 시작순 정렬 |
| `TraceSummarizer` | BOTTLENECK 30% 경계 위·아래 / 가장 이른 에러 span 선택 / exception event 없을 때 statusMessage / REPEATED_CALL 9건·10건 경계 / 아무것도 안 걸리면 빈 배열 |
| `TimeRange` | from > to → 400 / 24h 초과 → 400 / 경계값 |
| `TraceRepository` · `ServiceRepository` | Testcontainers ClickHouse에 고정 span 적재 후 쿼리 결과 검증. 랜덤 ServiceName으로 필터해 재실행 안전성 확보(apm-consumer 통합 테스트와 같은 방식) |
| `TraceController` | MockMvc — 없는 TraceId 404, 파라미터 검증 400, 응답 JSON 형태 |

### apm-web (Vitest + React Testing Library)

| 대상 | 검증 |
|------|------|
| `lib/layout.ts` | 진입 엣지 없는 노드가 레벨 0 / 순환 그래프에서 무한 루프 없음 |
| `Waterfall` | depth 들여쓰기, self/child 두 구간 폭, 에러 span 자동 펼침 |
| `TraceSummaryCard` | summary 빈 배열이면 렌더 안 함 |
| 페이지 | axios `jest.mock`(vi.mock) 고정 응답으로 로딩·에러·정상 |

목표 커버리지 80%.

---

## 11. 작업 순서

**Phase 1 — apm-api 조회 기반**
1. `apm-api` Gradle 프로젝트 생성, Spring Boot 3.3 / Java 21 / clickhouse-jdbc / springdoc / CorsConfig
2. `TimeRange` 검증 + `SpanSql` 상수
3. `GET /services`, `GET /service-map`

**Phase 2 — 트레이스 상세 (핵심)**
4. `SelfTimeCalculator`
5. `SpanTreeBuilder`
6. `TraceSummarizer`
7. `GET /traces/{traceId}`
8. `GET /traces` (검색)

**Phase 3 — 서비스 상세**
9. `GET /services/{name}/overview`
10. `GET /services/{name}/jvm`

**Phase 4 — apm-web**
11. Vite + React 19 + TS + Tailwind + react-query + recharts, Vite proxy
12. `TimeRangePicker` + 서비스 맵 (`lib/layout.ts`, `ServiceMap`)
13. **트레이스 상세** — `Waterfall`, `SpanDetailPanel`, `TraceSummaryCard`
14. 트레이스 검색 — `LatencyScatter` + 목록
15. 서비스 상세 — RED + 시계열 + 엔드포인트 표 + JVM

Phase 2가 이 UI의 목적이라 Phase 3보다 먼저 한다. Phase 4의 13번이 12·14번보다 먼저 와도 된다 — 트레이스 상세는 TraceId만 있으면 열린다.

---

## 12. 범위 제외

| 항목 | 이유 |
|------|------|
| 인증·인가 | 로컬 실행 전제. 공개 배포 시 필수 |
| 로그 신호 | Agent가 `otel.logs.exporter=none`이고 Collector도 Kafka로 안 보낸다 |
| Exemplar로 metrics → traces 점프 | 테이블에 컬럼은 있으나 화면 4개 밖 |
| 알림·임계값 | 조회 전용 |
| tail 샘플링 대응 | 현재 `parentbased_always_on` 전량 저장이라 span 직접 집계가 성립한다. 샘플링을 넣으면 요청 수 집계를 `otel_metrics_histogram` 기반으로 옮겨야 한다 |
| 서비스 맵 힘 기반 레이아웃 | 서비스 2개 |
| Grafana 대시보드 제거 | 임의 SQL 드릴다운 용도로 유지 |
