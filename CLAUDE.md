# TaskSyncBoard - Claude Code 가이드

## 1. 프로젝트 개요

- **프로젝트명**: TaskSyncBoard (사내 협업/이슈 관리 시스템, Jira 클론)
- **목적**: 학습 목적 프로젝트. `TASKSYNCBOARD_STEP_BY_STEP.md`를 따라 단계별로 구현 중

## 2. 기술 스택 (현재 Phase 1 기준)

| 항목 | 내용 |
|------|------|
| Language | Java 17 |
| Framework | Spring Boot 3.3.x |
| Build Tool | Gradle (Groovy DSL) |
| 현재 의존성 | Spring Web, Lombok, spring-boot-starter-test |

> **주의**: 위 의존성 외 추가 금지. 각 Phase에서 지정된 의존성만 추가할 것.

## 3. 프로젝트 구조 규칙

```
com.tasksyncboard
├── domain
│   └── {도메인명}
│       ├── entity
│       ├── repository
│       ├── service
│       ├── controller
│       └── dto
└── global
    ├── config
    ├── error
    ├── common
    ├── security
    ├── aop
    └── logging
```

- **베이스 패키지**: `com.tasksyncboard`
- **도메인 패키지**: `com.tasksyncboard.domain.{도메인명}.{계층}`
- **공통 패키지**: `com.tasksyncboard.global.{분류}`

## 4. 코딩 컨벤션

- **Lombok 사용**: `@Getter`, `@NoArgsConstructor(access = PROTECTED)`, `@Builder`, `@RequiredArgsConstructor`
- **의존성 주입**: 필드 주입(`@Autowired`) 금지, **생성자 주입만 사용**
- **DTO/Entity 분리**: DTO와 Entity를 반드시 분리하여 사용
- **주석**: 클래스와 메서드에 **한글 주석**으로 역할 설명 포함

## 5. 현재 진행 상태

| Phase | 내용 | 상태 |
|-------|------|------|
| Phase 1 | 프로젝트 초기 세팅 | 진행 중 |

**아직 추가하면 안 되는 것** (해당 Phase 도달 시 추가 예정):
- JPA / Spring Data JPA
- Spring Security
- Redis
- QueryDSL
- Flyway
- 기타 미지정 의존성

## 6. Claude Code 협업 규칙

- **git commit/push는 사용자가 직접 처리한다.** Claude는 파일 수정만 하고, `git add / commit / push` 명령을 자동으로 실행하지 않는다.

## 7. 빌드/실행 명령어

```bash
# 빌드
./gradlew build

# 실행
./gradlew bootRun

# 테스트
./gradlew test
```
