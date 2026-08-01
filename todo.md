# 이슈 #13 — 서비스 맵·서비스별 대시보드

브랜치: `feat/#13` / base: `develop`

## 목표

span 관계(`ParentSpanId → SpanId`)로 서비스 간 호출 그래프를 그리고, 템플릿 변수로 서비스별 상세를 본다.
추가 수집 없이 ClickHouse에 이미 있는 데이터만 쓴다.

## 설계

### Node Graph 프레임 구분 — 검증 완료

Grafana 12.4.3 번들 원문 확인 결과 2단계로 동작한다.

```js
// 1단계 — 포함 필터 (넷 중 하나만 맞으면 됨)
a.filter(o => o.meta?.preferredVisualisationType==="nodeGraph"
           || o.name==="nodes" || o.name==="edges"
           || o.refId==="nodes" || o.refId==="edges"
           || new DataFrameView(o).getFieldByName("id"))

// 2단계 — nodes/edges 분리
e.name === "edges" || e.fields.some(n => n.name === "source") ? edges.push(e) : nodes.push(e)
```

`refId`를 `nodes`·`edges`로 두면 grafana-clickhouse-datasource 4.19.0이 **프레임 이름도 같은 값으로 설정**해 양쪽 단계에서 안전하다(`/api/ds/query` 실측 확인).

2단계 코드에서 `e.name === "edges"`가 **먼저** 검사되므로 `refId`(=프레임 이름)가 1차 분리 키이고 `source` 필드는 2차다.

**`refId`를 바꾸면 안 된다.** 엣지 쿼리가 0행이면 플러그인이 `fields: []`인 프레임을 돌려주는데(실측),
이때는 `source` 필드 자체가 없어 `refId: "edges"`가 유일한 분리 근거가 된다.

`source` 별칭을 바꾸면 분리는 살아남지만 그 앞의 옵션 적용 함수가 `name` 검사 없이 `getFieldByName("source")`만 보므로,
엣지 프레임에 **노드용 unit과 arcs가 적용**되고 엣지 unit은 안 붙는다.

### arc 색상 — 패널 옵션에서만 지정된다

arc 색은 필드의 `config.color.fixedColor`에서 읽는데, 그 값을 채우는 경로는 **`options.nodes.arcs`** 하나뿐이다.
지정하지 않으면 `stroke=""`가 되어 테두리가 렌더되지 않는다. 에러율 0이면 `arc__success = 1`이라
"값이 1 이상이면 원 전체를 그 색으로" 분기를 타므로 **평상시에도 테두리가 통째로 안 보인다.**

```json
"options": {
  "nodes": {
    "mainStatUnit": "short",
    "secondaryStatUnit": "ms",
    "arcs": [
      { "field": "arc__error",   "color": "red" },
      { "field": "arc__success", "color": "green" }
    ]
  },
  "edges": { "mainStatUnit": "short", "secondaryStatUnit": "ms" },
  "layoutAlgorithm": "layered"
}
```

`mainStatUnit`/`secondaryStatUnit`도 함께 넣는다. 없으면 `v.toFixed(decimals || 2)`로 포맷돼 요청 수가 `101.00`으로 찍힌다.

기존 3개 대시보드는 `options`를 한 번도 쓰지 않았다 — 이 대시보드만의 예외다.

### 엣지 쿼리

```sql
SELECT
    concat(p.ServiceName, ' → ', c.ServiceName) AS id,
    p.ServiceName AS source,
    c.ServiceName AS target,
    count() AS mainstat,
    round(avg(c.Duration) / 1e6, 1) AS secondarystat
FROM otel.otel_traces AS c
INNER JOIN otel.otel_traces AS p
        ON c.TraceId = p.TraceId AND c.ParentSpanId = p.SpanId
WHERE c.SpanKind = 'Server'
  AND c.SpanName NOT LIKE 'GET /actuator%'
  AND c.SpanName NOT LIKE 'GET /swagger%'
  AND c.SpanName NOT LIKE 'GET /v3/api-docs%'
  AND c.Timestamp >= $__fromTime AND c.Timestamp <= $__toTime
  AND c.ServiceName != p.ServiceName
  AND p.Timestamp >= $__fromTime - INTERVAL 1 MINUTE AND p.Timestamp <= $__toTime
GROUP BY source, target
```

**`c.` 접두어를 전부 명시한다.** ClickHouse 25.3은 self-join에서 무접두어 컬럼에 `AMBIGUOUS_IDENTIFIER`를 내지 않고
**조용히 좌측 테이블에 바인딩**한다(실측). 지금은 의도대로 `c`에 걸리지만, `FROM ... AS p INNER JOIN ... AS c`로 순서만 바꿔도
필터가 에러 없이 부모 쪽으로 옮겨가고 `SpanKind='Server'`가 부모에 걸려 엣지가 0행이 된다.

**부모 하한을 1분 뒤로 미는 이유** — `Timestamp`는 span **시작 시각**이라 부모가 항상 자식보다 이르다.
양쪽에 같은 하한을 걸면 부모가 창 시작 직전에 시작한 호출이 통째로 사라진다. 실측 간격 0.9ms~49.6ms, 부모 Duration 9~261ms.

```
창 시작을 부모와 자식 사이에 두면
  양쪽 동일 필터  → count() = 0     ← 멀쩡한 엣지가 사라짐
  부모 하한 완화  → count() = 1
```

`now-5m` 같은 짧은 창에서 흔히 발생한다.

**성능 근거 (실측으로 정정)** — `EXPLAIN indexes=1` 결과 `idx_trace_id`는 목록에 나타나지 않는다.
bloom filter는 **상수 술어**(`TraceId = '리터럴'`)에만 적용되고 JOIN ON의 컬럼 대 컬럼 비교에는 걸리지 않는다.

| 조건 | 실제 효과 |
|------|-----------|
| 양쪽 시간 필터 | **파티션 프루닝** — 유일하게 읽는 행 수를 줄인다 |
| `c.TraceId = p.TraceId` | 조인 키 선택도. 결과 폭발 방지. I/O 감소 없음 |
| `c.SpanKind = 'Server'` | 읽은 뒤 필터링. 프로브 횟수 감소. I/O 감소 없음 |

`system.query_log` 실측: `read_rows = 1726` (= 전체 863 × 2), `query_duration_ms = 10`.
현재 규모에선 문제없다. 데이터가 커지면 오른쪽(`p`)을 `SpanKind = 'Client'` 서브쿼리로 감싸 해시 테이블을 줄인다 —
지금 넣지 않는 이유는 레거시 `SPAN_KIND_CLIENT` 값이 남아 있어 엣지가 누락될 수 있어서다(TTL로 2026-07-22 소멸 예정).

루트 span은 `ParentSpanId = ''`라 어떤 `SpanId`와도 매칭되지 않아 자연히 제외된다 — 진입점 노드는 범위 밖.

### 공통 WHERE 블록 — span 기반 RED 전부에 동일 적용

**두 대시보드의 모든 span 기반 쿼리가 같은 모집단을 써야 한다.** 노드 쿼리에만 노이즈 필터를 걸면
노드를 클릭해 상세로 넘어갈 때 같은 "요청 수"가 53 → 80으로 51% 뛴다 (실측).
http-performance ↔ service-detail 불일치는 근거 데이터가 달라 정상이지만, 이쪽은 둘 다 span 기반이라 불일치할 이유가 없다.

```sql
-- SPAN_RED_FILTER: service-map 노드·엣지·서비스 목록, service-detail의 stat·시계열·엔드포인트 테이블에 그대로 붙인다
SpanKind = 'Server'
AND SpanName NOT LIKE 'GET /actuator%'
AND SpanName NOT LIKE 'GET /swagger%'
AND SpanName NOT LIKE 'GET /v3/api-docs%'
AND Timestamp >= $__fromTime AND Timestamp <= $__toTime
```

- `SpanKind = 'Server'` — 서비스가 **받은** 요청만 센다. 전체 span을 세면 JPA·커밋 같은 내부 span 개수에 좌우돼 서비스 규모가 왜곡된다
- **노이즈 제외** — 실측상 Server span의 34%가 헬스체크·Swagger다 (apm-payment 101→67, apm-demo 80→53). 빼지 않으면 "요청 수"가 사실상 헬스체크 카운터가 된다
- 엣지 쿼리는 `c` 쪽에 붙인다. 서비스 간 호출에 헬스체크가 낄 일은 지금 없지만, 규칙이 갈리면 나중에 노드 합 < 엣지 합이 된다
- service-detail 쿼리는 여기에 `ServiceName = '$service'`를 추가한다

### 노드 쿼리

```sql
SELECT
    ServiceName AS id,
    ServiceName AS title,
    count() AS mainstat,
    round(avg(Duration) / 1e6, 1) AS secondarystat,
    countIf(StatusCode = 'Error') / count() AS arc__error,
    1 - countIf(StatusCode = 'Error') / count() AS arc__success
FROM otel.otel_traces
WHERE /* SPAN_RED_FILTER */
GROUP BY ServiceName
```

- `arc__` 합은 `1 - err`로 계산해 정확히 1이 보장된다 (실측: 세 서비스 모두 오차 0)
- `GROUP BY`라 그룹당 `count() >= 1`이 보장돼 0 나누기가 발생하지 않는다. 별칭 쌍따옴표도 불필요

**서비스 목록 테이블은 첫 컬럼을 별칭 없이 `ServiceName`으로 SELECT한다.**
노드 쿼리의 `AS id`를 복사하면 데이터 링크의 `${__data.fields.ServiceName}`이 빈 값으로 전개된다 —
`${__data.fields.X}`는 프레임의 **필드 이름**으로 해석되는데 노드 프레임에는 `ServiceName` 필드가 없다.

### 트래픽 0일 때의 NaN

`GROUP BY` 없는 집계는 **행이 0건이어도 1행을 반환**하고 그 값이 `0/0 = nan`이다.
Grafana는 "No data"가 아니라 `NaN`을 그린다 (`entities.NaN` 실측 확인, `lastNotNull` 리듀서로 못 거른다).

**`else 0`으로 막으면 안 된다.** NaN은 사라지지만 데이터가 없는 시간 범위에서 "정상"으로 읽히는 숫자가 나온다 —
계측이 죽어도 에러율 stat이 초록 0%, p95가 0ms를 띄운다. 이 이슈의 동기가 "계측 누락을 못 알아챘다"인데 정면으로 어긋난다.

**`NULL`을 반환하면 Grafana stat이 "No data"를 그린다** (`/api/ds/query`로 `nullable: true`, `values: [[null]]` 확인).

```sql
-- 요청 수: 0건은 참값이라 0 그대로
SELECT count() AS requests FROM otel.otel_traces WHERE /* SPAN_RED_FILTER */ AND ServiceName = '$service'

-- 에러율·p95: 트래픽 없으면 NULL
SELECT if(count() > 0, countIf(StatusCode = 'Error') / count(), NULL) AS error_rate ...
SELECT if(count() > 0, quantileExact(0.95)(Duration) / 1e6, NULL)     AS p95_ms ...
```

`quantileExact`는 빈 입력에서 `0`을 돌려주므로 방어 없이는 p95도 0으로 보인다.

단위는 **`percentunit`**(0~1 분수). `jvm.json`의 CPU 패널이 이미 `percentunit` + `max: 1`을 써서 일관적이다.
`*100` 후 `percent`를 쓰면 컨벤션이 갈린다.

stat 패널의 `reduceOptions.fields`를 비우면 프레임의 **모든** 숫자 필드가 표시되므로, 패널당 값 1개만 SELECT 한다.

### 백분위

`quantile`은 reservoir sampling 기반이라 같은 데이터에서도 값이 흔들린다. 새로고침마다 p95가 미세하게 바뀐다.
72시간 TTL·서비스당 수만 행 이하 규모에서는 **`quantileExact`** 가 비용 차이 없이 결정적이라 이쪽을 쓴다.

### span 기반 RED의 전제

기존 대시보드는 HTTP·JVM을 메트릭(`otel_metrics_*`) cumulative 차분으로 다룬다.
서비스 맵은 **span 기반**이다 — 서비스·엔드포인트 단위 집계가 직접 되고 차분이 필요 없다.

전제는 **전량 저장**이다. 레포에 sampler 설정이 없어 OTel Java Agent 기본값 `parentbased_always_on`이 적용된다(4개 레포 grep 확인).
로드맵의 tail 샘플링 단계를 하면 span이 일부만 남아 요청 수가 실제보다 작아지므로, 그때 RED를 메트릭 기반으로 옮겨야 한다.

### 기존 대시보드와의 역할 구분

`http-performance`와 `service-detail`의 엔드포인트별 테이블은 같은 질문에 다른 근거로 답한다. **숫자가 어긋나는 게 정상**이라 패널 설명과 README에 명시한다.

| | http-performance | service-detail |
|---|---|---|
| 근거 | `otel_metrics_histogram`, cumulative 차분 | `otel_traces` span 실측 |
| p95 | 버킷 상한 **근사** | 실측 분포 |
| 성격 | 공식 지표. 샘플링 무관 | 트레이스로 바로 연결됨 |

### 대시보드 구성

**`service-map.json`** (uid `apm-service-map`, `APM / 서비스 맵`)

| 패널 | 타입 | 내용 |
|------|------|------|
| 서비스 호출 그래프 | `nodeGraph` | 노드(요청수·평균지연·에러비율) + 엣지(호출수·평균지연) |
| 서비스 목록 | `table` | 요청수·에러율·p95. 서비스명 클릭 → 상세 |

**`service-detail.json`** (uid `apm-service-detail`, `APM / 서비스 상세`)

| 패널 | 타입 | 내용 |
|------|------|------|
| 요청 수 / 에러율 / p95 | `stat` ×3 | NaN 방어 적용 |
| 분당 요청·에러 | `timeseries` | span 기반, 1분 버킷 |
| 엔드포인트별 | `table` | `SpanName` 단위 요청수·p95·에러율 |
| 느린 트레이스 | `table` | 해당 서비스의 **진입 span**(`SPAN_RED_FILTER` + `ServiceName`). TraceId 클릭 → `apm-traces` |
| 힙 / 스레드 / CPU | `timeseries` ×3 | `jvm.json` 쿼리 + `ServiceName = '$service'` |

uid를 `apm-service-map`/`apm-service-detail`로 두는 이유: 기존은 `apm-http`·`apm-jvm`·`apm-traces`로 단어 하나지만, 이 둘은 링크로 이어진 한 쌍이라 접두를 공유하는 편이 관계가 드러난다.

**`trace-search.json`** 수정 — 도메인 관측 데이터 노출

- span 상세 패널에 도메인 키를 **컬럼으로** 추출. `SpanAttributes`를 통째로 SELECT하면 `db.statement` 등 10여 개가 한 셀에 JSON으로 들어가 도메인 값이 묻힌다 (Map은 없는 키에 빈 문자열을 돌려줘 안전)

```sql
SpanAttributes['order.id']     AS order_id,
SpanAttributes['order.status'] AS order_status,
SpanAttributes['payment.id']   AS payment_id
```

- **span event 패널 신규 추가**. apm-demo #7이 만드는 event 4종과 Agent의 `exception` event가 현재 어느 대시보드에도 보이지 않는다.

| event | 성격 |
|-------|------|
| `order.status.changed` | 상태 전이 이력. attribute는 덮어쓰기라 여기서만 보인다 |
| `order.payment.orphaned` | 결제 성사 후 확정 실패 |
| `order.payment.over-cancelled` | 결제 취소 후 주문 취소 실패 |
| `order.payment.outcome-unknown` | 타임아웃이라 결제 성사 여부 불명 |

아래 3종은 **응답이 409(4xx)이거나 502라도 원인이 구분되지 않아 `StatusCode`만으로는 판별할 수 없는 실패**다.
이 이슈의 동기("에러율에 안 잡히는 문제를 못 본다")와 성격이 정확히 같다.

`attrs`를 통째로 내보내면 안 된다 — 바로 위에서 `SpanAttributes`에 대해 금지한 것과 같은 문제다.
현재 DB의 event 56건 중 `exception`이 18건(32%)이고 그 `attrs` 한 셀이 **4KB 이상**(스택프레임 40여 줄)이라 행 하나가 화면을 덮는다.

```sql
SELECT
    ServiceName, SpanName,
    e.1 AS ts,
    e.2 AS event,
    e.3['order.id']          AS order_id,
    e.3['order.status']      AS order_status,
    e.3['payment.id']        AS payment_id,
    e.3['exception.type']    AS exc_type,
    e.3['exception.message'] AS exc_message
FROM otel.otel_traces
ARRAY JOIN arrayZip(Events.Timestamp, Events.Name, Events.Attributes) AS e
WHERE TraceId = '${trace_id}'
ORDER BY ts
```

`ServiceName`을 넣는 이유: 워터폴과 나란히 놓이는 패널인데 없으면 어느 서비스의 event인지 알 수 없다.
스택트레이스는 뺀다 — 필요하면 워터폴 패널에서 span을 눌러 본다.

**`http-performance.json`** 수정 — `ServiceName` 분리 (**현재 값이 틀리고 있다**)

이슈 개요가 문제 삼은 "대시보드가 전부 단일 서비스 기준"에 이 대시보드가 그대로 해당한다.
세 패널 모두 `Attributes['http.route']`로만 그룹핑하고 `ServiceName`이 없다(`series = toString(Attributes)`는 데이터포인트 attribute라 서비스명을 포함하지 않는다).
같은 레포의 `jvm.json`은 이미 전부 `ServiceName`으로 분리하고 있어 컨벤션이 갈라져 있다.

**가정법이 아니라 이미 발생 중인 버그다.** 실측상 네 개 route가 두 서비스에 공통으로 존재한다 —
`/actuator/health`, `/swagger-ui*/**`, `/v3/api-docs/swagger-config`, `/swagger-ui*/*swagger-initializer.js`.

```
같은 창, route = /swagger-ui*/**, status 200
  현재 쿼리 → requests = 7
  고친 쿼리 → apm-payment 6 + apm-demo 0 = 6
```

`GROUP BY t, route, series`로 뭉치면 두 서비스의 누적 카운터가 `max(Count)`로 한 값이 되어 차분이 틀린다.
`/actuator/health`는 10행이 6행으로 줄어 한 서비스 트래픽이 통째로 사라진다.
`ExplicitBounds` 길이가 두 서비스 모두 14로 같아 `sumForEach` 예외도 안 나고 **조용히 틀린 값만** 나온다.

`GROUP BY`에 `ServiceName`을 넣는 것만으로는 부족하다. 창 함수가 `PARTITION BY series`라
한 파티션 안에 두 서비스의 누적 카운터 행이 시간순으로 섞이고, `greatest(0, cnt - lagInFrame(cnt))`가
서비스 전환 지점마다 0 또는 스파이크를 만든다.

**중첩 SELECT의 모든 레벨을 고쳐야 한다.** `GROUP BY`와 `PARTITION BY`만 바꾸면 중간 레벨이 `ServiceName`을 넘겨주지 않아
`Code: 47. Unknown expression identifier 'ServiceName' in scope`로 컴파일에 실패한다(실행 확인).
최종 SELECT 출력 컬럼도 빠뜨리면 에러 없이 **같은 route가 서비스 수만큼 중복 라인**으로 그려진다.

각 패널에서 네 곳을 모두 손본다.

| 패널 | 중첩 | 고칠 곳 |
|------|------|---------|
| 1 (분당 요청 수) | 3단 | 모든 서브쿼리 SELECT 리스트 + 내부 `GROUP BY t, ServiceName, route, series` + `OVER (PARTITION BY ServiceName, series ORDER BY t)` + 외부 `GROUP BY t, ServiceName, route` + 최종 출력 컬럼 |
| 2 (평균 응답 시간) | 3단 | 위와 동일 + `WINDOW w AS (PARTITION BY ServiceName, series ORDER BY t)` |
| 3 (p50/p95/p99) | **4단** | 네 레벨 전부의 SELECT 리스트 + 최내부 `GROUP BY ServiceName, route, series` + 그 위 `GROUP BY ServiceName, route` + 최종 출력 컬럼 |

수정 후 실측 — 사라져 있던 행이 복원된다.

```
현재(깨진 쿼리)                고친 쿼리
/api/payments            6     apm-payment /api/payments             6
/api/orders             14     apm-demo    /api/orders              14
/swagger-ui*/**          3     apm-demo    /swagger-ui*/**           3
/api/payments/{id}/cancel 1    apm-payment /swagger-ui*/**           6   ← 통째로 사라져 있던 행
                               apm-payment /api/payments/{id}/cancel 1
```

이슈 작업 내용에 없지만 전제에 해당하고 현재 값이 틀리므로 범위에 포함한다.

**`jvm.json`** 수정 — GC 패널 2개에 같은 버그

패널 4(분당 GC 횟수)·6(평균 GC 소요)이 `PARTITION BY series ORDER BY t`를 쓰는데,
`series = toString(Attributes)`에 `ServiceName`이 없어 두 서비스의 GC 누적 카운터가 한 파티션에 시간순으로 섞인다.
http-performance와 **완전히 같은 메커니즘**이고, 현재 값이 실제의 약 2배로 부풀어 있다.

```
jvm.gc.duration series — 두 서비스가 동일 문자열
  apm-demo    {'jvm.gc.action':'end of minor GC','jvm.gc.name':'G1 Young Generation'}   961
  apm-payment {'jvm.gc.action':'end of minor GC','jvm.gc.name':'G1 Young Generation'}  1094

분당 GC 횟수 24h 합계
  현재            G1 Concurrent GC 30 / G1 Young Generation 45
  ServiceName 분리  12 + 4 = 16      /  28 + 7 = 35
```

같은 파일의 패널 1·2·3·5는 이미 `ServiceName`으로 분리돼 있어 컨벤션도 일치한다.
http-performance와 수정 패턴이 동일하므로 함께 처리한다.

### 데이터 링크

방향별로 문법이 다르다. `${service:queryparam}`은 **그 대시보드에 `service` 변수가 있을 때만** 전개된다.

| 방향 | URL |
|------|-----|
| 서비스 목록 → 상세 | `/d/apm-service-detail?var-service=${__data.fields.ServiceName}&${__url_time_range}` |
| 느린 트레이스 → 워터폴 | `/d/apm-traces?var-trace_id=${__data.fields.TraceId}&${service:queryparam}&${__url_time_range}` |

`service-map.json`에는 `$service` 변수를 두지 않는다(서비스 목록이 전 서비스를 나열하는 게 목적).
따라서 거기서 `${service:queryparam}`을 쓰면 빈 문자열이 되어 상세가 엉뚱한 서비스를 연다 — 행 값에서 뽑는 `${__data.fields.ServiceName}`을 써야 한다.

### `$service` 변수

Grafana에 대시보드 간 변수 공유는 없다. 각 대시보드가 자기 정의를 갖고 값 전달은 URL `var-service=`뿐이다.
쿼리는 `trace-search.json`과 동일하게 두고 시간 필터도 붙이지 않는다 — 기존 동작을 승계한다(TTL 72시간이라 폭주하지 않는다).

**JVM 패널은 조용히 빌 수 있다.** `$service` 옵션은 `otel_traces`에서 뽑는데 JVM 패널은 `otel_metrics_*`를 읽고, 두 테이블의 서비스 집합이 다르다.
실측: traces에는 `it-test`, metrics에는 `it-metrics-ba26e8c3f22d33f6`. 대시보드 `description`에 명시한다.

### 컨벤션

- datasource: `{ "type": "grafana-clickhouse-datasource", "uid": "clickhouse-otel" }`
- `editorType: "sql"`, `queryType`, `format` — nodeGraph는 전용 queryType이 없어 **`queryType: "table"`, `format: 1`**
- 시간 필터는 `$__timeFilter`가 아니라 **`Timestamp >= $__fromTime AND Timestamp <= $__toTime`**
- `schemaVersion: 39`, `editable: false`, `refresh: "30s"`, `tags: ["apm"]`

### 컨슈머 변환 규칙

| 컬럼 | 저장 값 |
|------|---------|
| `StatusCode` | `Ok` / `Error` / `Unset` |
| `SpanKind` | `Server` / `Client` / `Internal` / `Producer` / `Consumer` |
| `SpanAttributes` | `Map(LowCardinality(String), String)` — 값은 항상 문자열 |

대문자로 쓰면 0행이 나온다.

**레거시 값 주의** — 2026-07-20 00:19 이전 데이터에 변환 전 원본(`SPAN_KIND_SERVER` 등 22건)이 남아 있다. TTL로 2026-07-22 10:08 소멸 예정이며, E2E를 `now-1h` 같은 최근 창에서 하면 자연히 제외된다.

## 작업 목록

### 구현
- [x] `grafana/dashboards/service-map.json` — Node Graph(+`options.nodes.arcs`) + 서비스 목록
- [x] `grafana/dashboards/service-detail.json` — `$service` 기반 상세, stat NaN 방어
- [x] `grafana/dashboards/trace-search.json` — 도메인 attribute 컬럼 + span event 패널
- [x] `grafana/dashboards/http-performance.json` — 패널 3개 `ServiceName` 분리 (중첩 SELECT 전 레벨)
- [x] `grafana/dashboards/jvm.json` — GC 패널 4·6 `ServiceName` 분리 (같은 버그)
- [x] 패널·대시보드 `description` 작성 — 기존 3개 대시보드는 대시보드 레벨만 채워져 있고 **패널 레벨은 전부 빈 문자열**이라 관성대로 가면 누락된다
  - service-detail 엔드포인트 테이블 패널: http-performance와 숫자가 다른 이유(메트릭 근사 vs span 실측)
  - service-detail 대시보드: `$service`가 traces 기준이라 메트릭 없는 서비스는 JVM 패널이 빈다
- [x] `README.md` — 아래 3곳
  - 14행 grafana 서비스 행의 용도 서술(대시보드 5종)
  - 184~189행 대시보드 표에 신규 2종 추가 + 메트릭/span 기반 구분 명시
  - 190행 cumulative temporality 문단에 "메트릭 기반 대시보드에 한함" 한정어

### 마무리
- [x] JSON 문법·필드 규격 검증
- [ ] Critic 3회 검증
- [ ] `/code-review`

### E2E 수동 검증

전제 — apm-infra compose 기동 + apm-consumer 기동 + 두 서비스를 **Agent와 함께** 실행

```bash
cd ~/진기태/apm-kitae/apm-payment && ./scripts/run-with-agent.sh   # 18081
cd ~/진기태/apm-kitae/apm-demo    && ./scripts/run-with-agent.sh   # 18080
curl -X POST http://localhost:18080/api/orders -H "Content-Type: application/json" \
  -d '{"customerId":"c1","productId":"p1","quantity":2,"unitPrice":4500}'
```

대시보드 반영에는 `docker compose restart grafana` 필요 (`allowUiUpdates: false`).

- [ ] 시간 범위 `now-1h`에서 `apm-demo`·`apm-payment` 노드가 뜨고 그 사이 엣지 1건 이상
  (`it-test`가 함께 뜰 수 있다 — 통합테스트 잔여 데이터라 정상)
- [ ] **노드 테두리가 초록으로 칠해지는지** — `options.nodes.arcs` 누락 시 아무것도 안 그려진다
- [ ] mainstat이 `101.00`이 아니라 `101`로 표시되는지 (`mainStatUnit`)
- [ ] `APP_FAULT_ERROR_RATE=1`로 apm-payment 재기동 → 주문 호출 → **시간 범위 `now-2m`** 에서 노드 테두리에 빨강 비율 반영
  (창을 넓게 두면 앞 단계의 정상 트래픽에 희석돼 비율이 안 보인다)
- [ ] apm-payment를 Agent 없이 기동 → 주문 호출 → **시간 범위 `now-2m`** 에서 노드가 apm-demo만
  (TTL 72시간이라 창이 넓으면 과거 span 때문에 노드가 계속 뜬다 — 항상 실패한다)
- [ ] 서비스명 클릭 → 상세 대시보드가 해당 서비스로 열리는지
- [ ] `$service`를 `apm-payment`로 전환 → 패널이 해당 서비스만
- [ ] 트래픽 없는 시간 범위(예: `now-5m`에 요청 없음)에서 에러율·p95 stat이 `0`이 아니라 **No data**인지 (요청 수는 `0`이 정상)
- [ ] span 상세에서 `order_id`·`order_status`·`payment_id` 컬럼 확인
- [ ] span event 패널에서 `order.status.changed` 2건(PENDING·CONFIRMED) 확인
  - `orphaned`·`over-cancelled`·`outcome-unknown` 3종은 현재 DB에 한 건도 없다(재현 조건이 까다로움). 패널에 안 뜨는 게 정상이라 검증 대상에서 뺀다
  - **`order_id`·`payment_id` 컬럼이 비는 것도 정상.** `tagOrderTransition()`은 event에 `order.status`만 싣는다(소스 확인). 이 두 키는 위 3종 event에만 실린다. 값은 span **attribute** 쪽(`order_id` 19건·`payment_id` 17건 실측)에서 보인다
- [ ] http-performance에서 `/actuator/health` 같은 공통 route가 서비스별로 분리돼 보이는지
- [ ] 시간 범위 24시간에서 서비스 맵 로딩 **5초 이내**

## 구현 후 검증 이력

### Critic 3회 (병렬)

- **A (Grafana 렌더링)**: NEEDS_IMPROVEMENT — 4건 반영. 번들 원문 추출 기반
  - **MAJOR**: `mainStatUnit: "short"`가 노드에 `21.00 short`로 찍힌다. 스탯 포맷 함수가 `getValueFormat`을 안 거치고 unit ID를 그대로 이어붙인다. `"ms"`가 멀쩡해 보인 건 unit ID와 표기가 우연히 같아서였다
  - MINOR: 범례에 `arc__error`/`arc__success` 노출 → `displayName` override, 대시보드 설명이 노드 클릭으로 읽히는데 링크는 표에만 있음
- **B (쿼리 정확성)**: **OK** — 쿼리 20개 실행, 값 오류 0. service-map 21/6 == service-detail 21/6 일치 확인. MINOR 4건 중 3건 반영, 1건 기각
  - 기각: "`it-test`는 Server span이 0이라 패널이 빈다" → 실제로는 9건 있고 정상 표시된다
- **C (완결성·문서)**: NEEDS_IMPROVEMENT — 6건 반영
  - **MAJOR**: span event 패널 설명이 3종을 "409라 Error로 안 잡힌다"로 뭉갰는데 `outcome-unknown`은 502라 **Error로 기록된다** — 패널 존재 이유를 설명하는 문장에 반례가 들어갔다

### `/code-review` — REQUEST CHANGES → 반영 완료

- **HIGH**: 엣지 source가 노드 집합에 없으면 **Node Graph 패널이 통째로 죽는다**. Grafana의 `M()`이 `e[source].nodeRadius`를 null 가드 없이 읽어 `TypeError`가 난다.
  부모 하한 1분 완화가 만든 부작용이고, 인바운드 Server span이 없는 서비스(컨슈머·스케줄러)가 생기면 결정적으로 재현된다.
  → 엣지 source를 노드 집합으로 제한(`p.ServiceName IN (...)`). 실측: 재현 창에서 엣지 1 → 0, 정상 창에서는 그대로 6
- **MEDIUM**: `decimals: 0`이 무효(`0 || 2` → 2). `toString(count()) AS mainstat`으로 문자열 분기를 태워 해결
- **MEDIUM**: 템플릿 변수가 이스케이프 없이 SQL에 들어간다 → `${service:sqlstring}`·`${trace_id:sqlstring}` (신규 9곳 + 기존 5곳)

## 후속 이슈 후보

- **Grafana 전용 ClickHouse 계정 분리** — 현재 datasource가 쓰는 `apm` 계정에 `DROP`·`SOURCES`·`SYSTEM` 권한이 있다. `GRANT SELECT ON otel.*`만 가진 계정으로 분리
- **공통 WHERE 절 뷰로 추출** — 같은 필터가 9곳에 복제돼 있고 그중 한 곳은 `c.` 접두어 변형이다. `otel.span_red` 뷰를 만들면 정리되지만 기존 ClickHouse에 수동 생성이 필요해 이번 범위에서 뺐다
- **http-performance·jvm에 `$service` 변수** — `ServiceName` 분리로 시리즈가 서비스 배수로 늘었다. 지금은 16개지만 서비스가 10개가 되면 못 읽는다
- **`$service` 변수 쿼리 비용** — 시간 조건이 없어 30초마다 전 구간을 읽는다(`read_rows` 863). `refresh: 1`로 낮추는 편이 안전

## 알려진 한계

- **레거시 enum 값 언더카운트** — `SpanKind = 'Server'` 등호 비교라 구 컨슈머가 넣은 `SPAN_KIND_SERVER` 14건이 집계에서 빠진다(그중 apm-demo의 실제 요청 2건). 값이 틀리는 게 아니라 조용히 적게 센다. TTL로 2026-07-22 소멸하므로 그대로 둔다. 굳이 맞추려면 `SpanKind IN ('Server', 'SPAN_KIND_SERVER')`
- **엣지 쿼리에 중복 span 방어 없음** — Kafka at-least-once로 같은 span이 두 번 적재되면 엣지 `mainstat`이 배수로 뛴다. 노드 `count()`도 함께 뛰지만 배율이 달라 "노드 합 == 엣지 합"이 깨진다. DB에 이미 중복 SpanId가 1건 있으나(`299aae4d…`) `it-test`의 레거시 SpanKind라 위 필터에 걸려 지금은 안 보인다 — **두 한계가 서로를 가리고 있다.** 근본 대응은 `ReplacingMergeTree` 또는 엣지 서브쿼리에서 `p`를 `SELECT DISTINCT`로 감싸는 것

## 범위 제외

진입점(사용자/브라우저) 노드, 서비스 맵 자동 이상 탐지, 노드 위치 고정(`fixedX`/`fixedY`), 알림 규칙,
`jvm.json`의 GC 패널을 service-detail로 옮기는 것(값 수정은 범위에 포함하되 이관은 하지 않는다)

## 검증 이력

### 1차 (3개 병렬)

- **A (Grafana)**: NEEDS_IMPROVEMENT — 7건. Grafana 12.4.3 번들 원문과 `/api/ds/query` 실측 기반
  - 최대 수확: **arc 색상은 `options.nodes.arcs`에서만 지정된다.** 누락 시 테두리가 렌더되지 않고, 에러율 0이면 평상시에도 안 보인다 — E2E 기준이 그대로 실패했을 것
  - 프레임 구분 메커니즘 정정: refId는 포함 필터용이고 실질 분리 키는 `source` 필드. 리스크가 과대평가돼 있었다
  - 트래픽 0일 때 stat이 `NaN`(entities.NaN 실측)
- **B (ClickHouse)**: NEEDS_IMPROVEMENT — 8건. 실제 쿼리 실행 기반
  - 최대 수확: **양쪽 시간 필터가 창 경계에서 엣지를 누락시킨다.** 부모 Timestamp가 시작 시각이라 자식보다 이르다
  - bloom filter 근거 오류 (`EXPLAIN indexes=1`에 `idx_trace_id` 미출현, `read_rows=1726`=전량)
  - 노이즈 34%, 레거시 enum 22건, `it-test` 서비스, `quantile` 비결정성
- **C (커버리지)**: NEEDS_IMPROVEMENT — 9건
  - 최대 수확: **#7의 span event 4종이 어느 대시보드에도 안 보인다.** 3종은 4xx라 에러율에도 안 잡혀 이 이슈의 동기와 성격이 같다
  - `http-performance`가 `ServiceName` 미분리 — 이슈가 문제 삼은 전제에 해당
  - 지표 중복 시 사용자 혼란, 데이터 링크 규격, README 갱신 대상 3곳

### 2차 (개정본 대상)

- **2차**: NEEDS_IMPROVEMENT — BLOCKER 1 / MAJOR 3 / MINOR 4, 전부 반영
  - **BLOCKER**: http-performance의 route 충돌이 가정법이 아니라 **이미 발생 중인 버그**였다. 네 route가 두 서비스 공통이고 `/swagger-ui*/**`가 현재 7 vs 실제 6, `/actuator/health`는 10행이 6행으로 줄어 한 서비스 트래픽이 사라진다. 처방도 부족했다 — `GROUP BY` 추가만으로는 `PARTITION BY series` 안에서 두 서비스 카운터가 섞여 차분이 깨진다. 패널 3개별 수정 지점을 표로 확정
  - **MAJOR**: 노이즈 필터를 노드 쿼리에만 걸어 노드→상세 이동 시 요청 수가 53→80으로 뛴다 → 공통 WHERE 블록으로 격상
  - **MAJOR**: span event 패널의 `attrs`가 4KB JSON 덩어리(exception이 event의 32%) → 도메인 키·예외 요약만 컬럼 추출 + `ServiceName` 추가
  - **MAJOR**: NaN 방어의 `else 0`이 "트래픽 없음"을 "에러율 0%·p95 0ms"로 표시 → 계측이 죽어도 초록으로 보인다. 에러율·p95는 `NULL`로 바꿔 No data가 뜨게
  - MINOR: 프레임 분리 설명 자기모순(`name`이 먼저 검사됨), 서비스 목록 컬럼 별칭, event 이름 3종 오기·재현 불가, 엣지 쿼리 노이즈 필터

  실측 통과 확인: `options.nodes.arcs` 중첩 구조·`layoutAlgorithm: "layered"` 유효, 노이즈 필터 후에도 arc 합 1, 부모 하한 1분 완화 엣지 쿼리 정상 전개, `arrayZip` + `ARRAY JOIN` 동작, `quantileExact` 결정적, provisioning 자동 로드

### 3차

- **3차**: NEEDS_IMPROVEMENT — MAJOR 2 / MINOR 4, 전부 반영
  - **MAJOR**: http-performance 수정 지점 표가 불완전. 패널 3은 4단 중첩인데 3곳만 지목해 그대로 적용하면 `Code: 47. UNKNOWN_IDENTIFIER`로 죽는다. 패널 1·2도 최종 출력 컬럼이 빠지면 에러 없이 같은 route가 중복 라인으로 그려진다 → "각 서브쿼리 SELECT 리스트 + GROUP BY + PARTITION BY + 최종 출력" 4항목으로 통일
  - **MAJOR**: `jvm.json` GC 패널 4·6에 **완전히 같은 버그가 이미 있다.** 범위 제외에 "옮기려면 구조 변경 필요"로 적어 "jvm은 안 틀렸다"로 오도했다. 실측상 현재 값이 실제의 약 2배(45 vs 35, 30 vs 16) → 범위에 포함
  - MINOR: 엣지 쿼리 무접두어 컬럼이 좌우 순서에 암묵 의존(ClickHouse가 조용히 좌측 바인딩) → `c.` 전부 명시, E2E 창 미지정으로 항상 실패하는 항목 2건, `description` 작업 항목 누락, event의 `order_id`·`payment_id`가 항상 빈 값인 것이 정상임을 미명시

  실측 통과 확인: `SPAN_RED_FILTER`를 `c` 쪽에 펼쳐 실행 → `apm-demo → apm-payment / 6 / 98.4ms` 정상, NULL 방어 `Nullable(Float64)` 동작, event 이름 4종이 `DomainSpanAttributes` 상수와 일치, README 행 번호 정확
