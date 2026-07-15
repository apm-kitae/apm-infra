# apm-infra

apm-kitae 인프라 실행 구성. Docker Compose로 로컬 개발/데모 환경의 인프라 컨테이너를 관리한다.

## 서비스 목록

| 서비스 | 이미지 | 포트 | 용도 |
|--------|--------|------|------|
| mysql | mysql:8.0 | 3306 | apm-demo 로컬 개발용 DB |

> 이후 추가 예정: OTel Collector, Kafka, ClickHouse, MinIO, Grafana

## 실행

```bash
cp .env.example .env      # 최초 1회, 필요 시 값 수정
docker compose up -d
docker compose ps         # 상태 확인 (healthy)
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

> `MYSQL_USER`/`MYSQL_PASSWORD`/`MYSQL_DATABASE`는 볼륨이 비어 있을 때 최초 1회만 적용된다. 값 변경 시 `docker compose down -v` 후 재기동.

## 접속 확인

```bash
mysql -h127.0.0.1 -P3306 -uapm -p1234 apm_demo
```

## 컨벤션

공통 개발 컨벤션: [apm-kitae.md](./apm-kitae.md)
