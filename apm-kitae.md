# apm-kitae 공통 컨벤션

> apm-kitae 조직의 모든 레포에 공통 적용되는 개발 컨벤션.
> 각 레포는 이 문서를 기준으로 하되, 레포 특화 내용은 각자의 README/CLAUDE.md에 기록한다.

---

## 기술 스택 (공통 고정)

| 영역 | 기술 | 비고 |
|------|------|------|
| 언어/런타임 | **Java 21 (LTS, 고정)** | 로컬 · CI · Docker 이미지 모두 동일 버전. Gradle toolchain으로 강제 |
| 프레임워크 | Spring Boot 3.3.x | |
| 빌드 | Gradle | `java.toolchain.languageVersion = 21` |
| API 문서 | springdoc-openapi (Swagger UI) | `/swagger-ui/index.html` |
| 테스트 | JUnit 5, Mockito, Testcontainers, H2 | 커버리지 80% 이상 목표 (JaCoCo) |
| 배포 | Docker Compose | |

## Git 컨벤션

### 브랜치
- 기본 브랜치: `develop` (PR base 기본값)
- 작업 브랜치: `{유형}/#이슈번호` — 예: `feat/#42`, `fix/#15`, `chore/#3`
- 유형: `feat` `fix` `refactor` `chore` `docs` `design` `test`

### 커밋 메시지
- 형식: `[유형] 작업 내용 (#이슈번호)` (한국어)
- 예: `[feat] 주문 조회 API 구현 (#42)`, `[chore] Docker MySQL 설정 추가 (#3)`

### 커밋 분리 원칙
- API 구현은 엔드포인트별로 커밋 분리
- 테스트는 구현과 분리해 `[test]` 커밋으로 작성
- Swagger 문서 변경은 별도 `[docs]` 커밋
- 공통 설정/유틸은 별도 커밋으로 먼저 분리
- 순서: 공통 설정/유틸 → 핵심 구현 → 테스트 → 문서

### 이슈/PR
- 이슈는 Phase(기능 단위)로 크게, 세부 체크리스트는 `todo.md`에서 관리
- 이슈 = "왜 이 작업을 하는가", 커밋 = "뭘 바꿨는가"
- refactor 이슈/PR은 문제점 근거 필수(수치, 코드 예시), fix는 원인 섹션 필수
- PR 구조: Summary / 변경 사항 테이블 / 테스트 결과 / 영향 범위

## 개발 원칙

- **TDD**: RED(실패 테스트) → GREEN(최소 구현) → REFACTOR. 테스트 먼저 작성
- 커밋 전 체크리스트:
  - [ ] `./gradlew test` 전체 통과
  - [ ] Swagger 문서 변경 시 `[docs]` 커밋 분리

## 빌드 & 테스트

```bash
./gradlew build          # 빌드
./gradlew test           # 테스트 실행
```

## 커밋/PR/이슈 서술 규칙 (절대 규칙 — 최우선)

- 커밋 메시지·PR 본문·이슈 본문은 **현재 코드가 무엇인지만** 서술한다
- **작업 이력 서술 절대 금지**: "전에 ~했다가 변경", "기존 ~에서 ~로 수정", "~를 ~로 재구성/이식" 같은 이전 작업 대비 표현을 쓰지 않는다. 작업 도중 몇 번을 갈아엎었든 결과물만 서술한다
  - 예외: fix의 "원인" 섹션, refactor의 "문제점 근거"는 현재 코드의 문제를 서술하는 것이므로 유지
- **추상어·비유어 절대 금지**: "골격", "뼈대", "건설", "기틀", "기반 마련", "초석", "부착" 같은 개발자가 실제로 쓰지 않는 비유적 표현을 쓰지 않는다. 클래스명·설정·엔드포인트 등 구체적 기술 명사로만 서술한다
  - ❌ "프로젝트 골격 구축" → ⭕ "Gradle 프로젝트 생성, Spring Boot 3.3 설정, Docker MySQL 구성"
  - ❌ "Agent 부착" → ⭕ "Agent 설정" — 대체어가 애매한 동작 표현은 "설정"으로 쓴다
- 이 규칙은 커밋/PR/이슈뿐 아니라 **Claude의 대화 답변에도 동일하게 적용**된다
