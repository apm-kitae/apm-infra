# Kafka 적재 데이터 읽기 문서

> Kafka 토픽에 실제로 들어간 텔레메트리 데이터를 처음 보는 사람을 위한 해설.
> 이 레포에서 적재 검증 때 나온 **실제 출력**을 그대로 까서 한 줄씩 읽는다.
> Collector debug 로그 읽는 법은 [OTel Log 읽기 문서](./OTel-Log-읽기-문서.md),
> 토픽·파티션 설계 근거는 [Kafka 설계 문서](./Kafka-설계-문서.md) 참고.

---

## 1. 큰 그림 — 데이터가 어디를 거쳐 어느 토픽에 앉는가

```
앱 (OTel Java Agent)
  │  OTLP http/protobuf (4318)
  ▼
OTel Collector
  │  traces 파이프라인  ──→ Kafka 토픽 traces   (키 = trace_id)
  │  metrics 파이프라인 ──→ Kafka 토픽 metrics  (키 없음)
  │  logs 파이프라인    ──→ debug 출력만 (토픽 미전송)
  ▼
Kafka (KRaft 단일 노드, 토픽당 파티션 3개)
```

기억할 것 하나: **debug 로그로 눈에 봤던 그 데이터가, 사람용 텍스트 대신 기계용
직렬화(Protobuf)로 바뀌어 토픽에 앉는다.** 내용은 동일하다.

---

## 2. 토픽 상태 확인 — 실제 출력 읽기

```bash
docker exec apm-kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --describe --topic traces
```

실제 출력:

```
Topic: traces  TopicId: w1UjqXyYSZ2yDD1vZYTSUA  PartitionCount: 3  ReplicationFactor: 1
    Topic: traces  Partition: 0  Leader: 1  Replicas: 1  Isr: 1
    Topic: traces  Partition: 1  Leader: 1  Replicas: 1  Isr: 1
    Topic: traces  Partition: 2  Leader: 1  Replicas: 1  Isr: 1
```

| 항목 | 값 | 뜻 |
|------|-----|-----|
| `PartitionCount: 3` | 파티션 3개 | 이 토픽의 병렬 차선 수. `KAFKA_NUM_PARTITIONS: 3` 설정으로 첫 메시지 수신 시 자동 생성됨 |
| `ReplicationFactor: 1` | 복제본 1 | 원본 1벌만 존재 (단일 브로커 데모라서. 프로덕션은 3) |
| `Leader: 1` | 브로커 1번 | 각 파티션의 쓰기 입구. 브로커가 1대뿐이라 전부 1번 |
| `Isr: 1` | 동기화된 복제본 | 복제본이 리더 자신뿐 |

파티션별 쌓인 양은 오프셋으로 본다:

```bash
docker exec apm-kafka /opt/kafka/bin/kafka-get-offsets.sh \
  --bootstrap-server localhost:9092 --topic traces
```

실제 출력:

```
traces:0:0     ← 파티션 0에 0건
traces:1:1     ← 파티션 1에 1건
traces:2:12    ← 파티션 2에 12건
```

형식은 `토픽:파티션:다음에 기록될 오프셋`(= 지금까지 쌓인 개수)이다.
분포가 0/1/12로 쏠려 있는데, 이는 **표본이 13건뿐이라 생기는 우연**이다 —
`hash(trace_id) % 3`은 trace_id가 많아질수록 균등에 수렴한다 (동전 3번 던져서
앞면만 나오는 것과 같은 소표본 현상).

---

## 3. 메시지 한 건 까보기 — 실제 소비 출력

```bash
docker exec apm-kafka /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic traces --from-beginning \
  --property print.key=true --property print.partition=true
```

테스트 span 1건을 보냈을 때의 실제 출력 (바이너리는 `.`으로 표시):

```
Partition:2  5b8efff798038103d269b633813fc60c  z.R.Az.A..3P.PR...*.kafka-smoke"...
                                               .manual-test ... .kafka-smoke-test
                                               .service.name ...
```

세 부분으로 나뉜다:

### ① `Partition:2` — 어느 차선에 앉았나

프로듀서(Collector)가 `hash(trace_id) % 3 = 2`를 계산해서 붙인 결과.
같은 trace_id는 언제나 파티션 2로 온다.

### ② `5b8efff798038103d269b633813fc60c` — 메시지 키

**trace_id 그 자체**(16바이트를 hex로 표기한 32자리)다.
collector 설정의 `partition_traces_by_id: true`가 "trace_id를 키로 세팅하라"는
스위치이고, 이 키가 ①의 해시 입력이 된다.
→ 같은 트레이스의 span들은 전부 같은 키 = 같은 파티션 = 같은 컨슈머.

### ③ 나머지 바이너리 — 메시지 값 (OTLP Protobuf)

깨진 문자 사이로 `kafka-smoke`(span 이름), `manual-test`(스코프),
`kafka-smoke-test`(service.name), `service.name`(속성 키)이 **평문으로 보인다.**

왜 반쯤 읽히는가: Protobuf는 **문자열은 그대로, 숫자·구조 정보는 바이트로**
직렬화한다. 그래서 이름·속성 같은 문자열은 눈에 보이고, trace_id·타임스탬프·
필드 구분자는 깨져 보이는 것이다. 이 값의 정체는:

```
ExportTraceServiceRequest 를 직렬화한 바이트
└── ResourceSpans[]                  ← debug 로그에서 본 3층 구조 그대로
    ├── Resource (service.name=kafka-smoke-test, ...)
    └── ScopeSpans[] (scope=manual-test)
        └── Span[] (name=kafka-smoke, traceId, kind, start/end time, ...)
```

**debug 로그와 내용이 동일하고 포장만 다르다** — 사람용으로 풀어 쓴 게 debug
출력, 기계용으로 압축한 게 이 바이너리다.

---

## 4. 예상 밖 손님 읽기 — service.name으로 발신자 구분

같은 토픽에서 실제로 함께 소비된 다른 메시지:

```
Partition:?  0a23bba0e6a681a7edd55f6fc654c28a  ... .evictExpired .code.function
             ... com.slunch.veggieverse.subscription.customplan.service.CustomPlanService
             ... .slunch-backend .service.name ...
```

읽어보면: span 이름 `evictExpired`, 클래스 `CustomPlanService`,
그리고 **`service.name = slunch-backend`** — apm-demo가 아니라 같은 호스트에서
돌던 다른 앱(slunch-backend)의 스케줄러 작업 span이다. 그 앱도 OTel Agent가
붙은 채 4318로 전송 중이어서 같은 Collector → 같은 토픽으로 흘러들어온 것.

여기서 배울 것 두 가지:

1. **토픽은 앱별이 아니라 신호별**이다 — 여러 앱의 트레이스가 한 토픽에 섞이는
   게 정상이고, 구분은 데이터 안의 `service.name`으로 한다
   ([OTel Log 읽기 문서](./OTel-Log-읽기-문서.md) 체크포인트 1번과 같은 원리)
2. 특정 앱만 받고 싶으면 Collector의 filter processor로 거른다 (별도 이슈 예정)

---

## 5. 전송 시나리오 — 주문 생성 1건이 토픽에 앉기까지

Swagger에서 `POST /api/orders`를 1번 눌렀을 때 일어나는 일의 전체 순서:

```
① 앱에서 span 5개 발생 (SERVER → save → persist → INSERT → commit)
   전부 같은 trace_id (예: 6941cd3b...)

② Agent가 몇 초간 모았다가(배치) OTLP로 Collector에 전송

③ Collector batch processor가 다시 묶음

④ kafka exporter가 처리:
   - trace_id별로 span을 묶어 ExportTraceServiceRequest 생성
   - Protobuf 직렬화 → zstd 압축
   - 키 = trace_id 세팅, hash(6941cd3b...) % 3 = 1 → "파티션 1행" 확정

⑤ 브로커의 파티션 1 리더가 받아서 로그 파일 끝에 추가
   → 오프셋 하나 증가 (traces:1:1 → traces:1:2)
```

metrics는 같은 흐름에서 두 가지만 다르다:
**키가 없어서**(메트릭엔 트레이스 같은 묶음 개념이 없음) 라운드로빈으로 파티션에
분배되고, 값의 타입이 `ExportMetricsServiceRequest`다. Agent가 60초 주기로
보내므로 요청이 없어도 꾸준히 쌓인다.

---

## 6. 이 데이터의 다음 여정 — 컨슈머 미리보기

토픽에 앉은 메시지는 컨슈머(Spring Boot 앱)가 이렇게 꺼내 읽게 된다:

```java
// Kafka 컨슈머는 ByteArrayDeserializer로 바이트를 그대로 받는다
ExportTraceServiceRequest request = ExportTraceServiceRequest.parseFrom(record.value());
// → 3절에서 본 바이너리가 객체로 복원됨. 이후 필드 추출 → ClickHouse 배치 INSERT
```

즉 이 문서에서 눈으로 깐 구조(키 = trace_id, 값 = ResourceSpans 트리)가
그대로 컨슈머 구현의 명세다. 뽑아낼 컬럼 후보는
[OTel Log 읽기 문서 7절](./OTel-Log-읽기-문서.md)의 표와 동일하다.

---

## 7. 직접 확인하는 명령 모음

```bash
# 토픽 목록
docker exec apm-kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --list

# 파티션별 쌓인 개수
docker exec apm-kafka /opt/kafka/bin/kafka-get-offsets.sh \
  --bootstrap-server localhost:9092 --topic traces

# 실시간 감시 (키·파티션 표시) — 이 상태에서 주문 API를 호출해보면
# Partition:N + trace_id 32자리 + 바이너리가 찍힌다
docker exec apm-kafka /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic traces \
  --property print.key=true --property print.partition=true

# 과거분부터 전부 보기: --from-beginning 추가 (로그라서 소비돼도 남아 있음)
```

## 체크포인트 요약

| 확인 항목 | 보는 곳 | 정상 기준 |
|----------|--------|----------|
| 토픽이 생겼는가 | `--list` | `traces`, `metrics` |
| 파티션 구성 | `--describe` | PartitionCount 3 |
| 데이터가 쌓이는가 | `kafka-get-offsets` | 오프셋 합이 증가 |
| 키 파티셔닝 동작 | 컨슈머의 key 출력 | 32자리 hex (trace_id) |
| 내 앱 데이터인가 | 바이너리 속 평문 | `service.name` 위치에 `apm-demo` |
| 값이 OTLP인가 | 바이너리 속 평문 | span 이름·스코프·속성 키가 보임 |
