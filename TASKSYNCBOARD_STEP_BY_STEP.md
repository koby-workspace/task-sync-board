# TaskSyncBoard - Spring Boot 3.x 단계별 실습 가이드

> **프로젝트명:** TaskSyncBoard (사내 협업/이슈 관리 시스템, Jira 클론)
> **이 문서는 단계별로 진행하며, 각 Phase에서 필요한 라이브러리와 개념을 그때그때 설명합니다.**

---

## 목차

- [Phase 1: 프로젝트 초기 세팅](#phase-1-프로젝트-초기-세팅)
- [Phase 2: Docker Compose 개발 환경 (PostgreSQL + Redis)](#phase-2-docker-compose-개발-환경-postgresql--redis)
- [Phase 3: 도메인 모델링 + JPA 딥다이브 + Flyway](#phase-3-도메인-모델링--jpa-딥다이브--flyway)
- [Phase 4: API 개발 + Validation + 전역 예외 처리](#phase-4-api-개발--validation--전역-예외-처리)
- [Phase 5: Spring Security + JWT + Redis 인증](#phase-5-spring-security--jwt--redis-인증)
- [Phase 6: AOP 마스터 - 이력 추적 및 권한 체크](#phase-6-aop-마스터---이력-추적-및-권한-체크)
- [Phase 7: QueryDSL + N+1 문제 해결](#phase-7-querydsl--n1-문제-해결)
- [Phase 8: 동시성 제어 (낙관적 락 / 비관적 락)](#phase-8-동시성-제어-낙관적-락--비관적-락)
- [Phase 9: 테스트 전략 (JUnit 5 + Mockito + Testcontainers)](#phase-9-테스트-전략-junit-5--mockito--testcontainers)
- [Phase 10: Swagger (Springdoc OpenAPI) 문서화](#phase-10-swagger-springdoc-openapi-문서화)
- [Phase 11: Actuator 모니터링 + Docker 이미지 빌드](#phase-11-actuator-모니터링--docker-이미지-빌드)
- [부록 A: 커스텀 로깅 전략](#부록-a-커스텀-로깅-전략)
- [부록 B: ERD 및 도메인 상세 설계](#부록-b-erd-및-도메인-상세-설계)

---

## 전체 로드맵 요약

각 Phase는 이전 Phase의 결과물 위에 쌓이며, **필요한 시점에 라이브러리를 설치**한다.

```
Phase 1: 프로젝트 뼈대 세팅 (Spring Web + Lombok만)
    │
Phase 2: Docker Compose (PostgreSQL + Redis 로컬 환경)
    │  ← DB가 있어야 JPA를 실습할 수 있다
    │
Phase 3: 도메인 모델링 + JPA + Flyway  ← ★ JPA, PostgreSQL Driver, Flyway 추가
    │  ← 엔티티와 테이블이 있어야 API를 만든다
    │
Phase 4: REST API + Validation + 전역 예외 처리  ← ★ Validation 추가
    │  ← 기본 CRUD가 동작해야 Security를 입힌다
    │
Phase 5: Spring Security + JWT + Redis 인증  ← ★ Security, JWT, Redis 추가
    │  ← 인증이 있어야 권한 체크(AOP)를 구현한다
    │
Phase 6: AOP 마스터 (이력 추적, 권한 체크 커스텀 어노테이션)
    │  ← 핵심 비즈니스 로직이 완성된 위에 횡단 관심사를 얹는다
    │
Phase 7: QueryDSL + N+1 문제 해결  ← ★ QueryDSL 추가
    │  ← 복잡한 조회 요구사항을 효율적으로 해결한다
    │
Phase 8: 동시성 제어 (낙관적 락 / 비관적 락)
    │  ← 여러 사용자가 동시에 수정하는 상황을 제어한다
    │
Phase 9: 테스트 (JUnit 5 + Mockito + Testcontainers)  ← ★ Testcontainers 추가
    │  ← 완성된 기능을 단위/통합 테스트로 검증한다
    │
Phase 10: Swagger (Springdoc OpenAPI) 문서화  ← ★ Springdoc OpenAPI 추가
    │
Phase 11: Actuator 모니터링 + Docker 이미지 빌드  ← ★ Actuator 추가
```

### TaskSyncBoard 핵심 도메인

| 도메인 | 설명 |
|--------|------|
| **Member** | 사용자 (회원가입, 로그인, 역할: ADMIN/MANAGER/MEMBER) |
| **Project** | 프로젝트 (이슈들을 그룹핑하는 워크스페이스) |
| **ProjectMember** | 프로젝트별 멤버 매핑 (역할: LEADER/MAINTAINER/DEVELOPER/VIEWER) |
| **Issue** | 이슈 티켓 (제목, 설명, 타입, 우선순위, 상태, 담당자, 보고자) |
| **IssueHistory** | 이슈 변경 이력 (AOP로 자동 기록) |
| **Comment** | 이슈에 달린 코멘트 |
| **Label** | 이슈 라벨/태그 |

---

# Phase 1: 프로젝트 초기 세팅

> **이번 Phase의 목표:** Spring Boot 프로젝트를 처음부터 만들고, 실행까지 해본다. 아직 어려운 개념은 신경 쓰지 않아도 된다. "일단 돌아가는 것"을 먼저 만들고, 그 다음에 "이게 뭐였지?"를 하나씩 이해하자.

## 1.1 시작하기 전에 준비할 것

아래 3가지가 컴퓨터에 설치되어 있어야 한다. 아직 없다면 먼저 설치하자.

| 준비물 | 설명 | 확인 방법 |
|--------|------|-----------|
| **JDK 17** | Java 프로그램을 만들고 실행하기 위한 도구 | 터미널에서 `java -version` 입력 → 17.x.x 나오면 OK |
| **IntelliJ IDEA** | Java 개발에 가장 많이 쓰이는 편집기(IDE). Community(무료) 버전도 충분하다 | 설치 후 실행되면 OK |
| **Claude Code** | AI 코딩 도우미. 터미널에서 명령어로 사용한다 | 터미널에서 `claude --version` 입력 → 버전 나오면 OK |

> **IDE가 뭔가요?** "통합 개발 환경(Integrated Development Environment)"의 줄임말이다. 코드 작성, 실행, 디버깅을 한 프로그램에서 다 할 수 있게 해주는 도구다. 메모장으로도 코드를 쓸 수 있지만, IDE를 쓰면 오타를 자동으로 잡아주고, 코드 자동완성도 해준다.

### IntelliJ Lombok 플러그인 확인

이 프로젝트에서 Lombok이라는 도구를 사용하는데, IntelliJ에서 제대로 동작하려면 설정이 필요하다.

1. IntelliJ 실행
2. `File` → `Settings` (Mac은 `IntelliJ IDEA` → `Preferences`)
3. 왼쪽 메뉴에서 `Plugins` 클릭
4. "Lombok" 검색 → 이미 설치되어 있으면 OK, 아니면 `Install` 클릭
5. `Settings` → `Build, Execution, Deployment` → `Compiler` → `Annotation Processors`
6. **"Enable annotation processing"** 체크 → `Apply`

> **Lombok이 뭔가요?** Java는 원래 코드가 좀 길어지는 편이다. 예를 들어 변수 하나의 값을 꺼내려면 `getXxx()` 메서드를 직접 만들어야 한다. Lombok은 `@Getter`라는 한 줄만 붙이면 이런 반복 코드를 자동으로 만들어준다. 자세한 건 코드를 작성하면서 하나씩 알게 된다.

## 1.2 IntelliJ에서 Spring Boot 프로젝트 만들기

### 방법 A: IntelliJ에서 직접 만들기 (추천)

1. IntelliJ 실행 → `New Project` 클릭
2. 왼쪽에서 **Spring Initializr** 선택
3. 아래와 같이 입력:

| 항목 | 입력값 |
|------|--------|
| Name | task-sync-board |
| Location | 원하는 폴더 경로 |
| Language | Java |
| Type | **Gradle - Groovy** |
| Group | com.tasksyncboard |
| Artifact | task-sync-board |
| Package name | com.tasksyncboard |
| JDK | 17 |
| Java | 17 |
| Packaging | Jar |

4. `Next` 클릭
5. Spring Boot 버전: **3.3.x** (최신 안정 버전) 선택
6. Dependencies(의존성) 선택 — **이 2개만** 체크하고 나머지는 건드리지 않는다:
   - **Spring Web** (웹 서버 기능)
   - **Lombok** (반복 코드 줄여주는 도구)
7. `Create` 클릭

> **의존성(Dependency)이 뭔가요?** 다른 사람이 미리 만들어둔 코드 묶음(라이브러리)이다. "Spring Web"을 추가하면 웹 서버를 만드는 데 필요한 코드를 가져다 쓸 수 있다. 처음부터 직접 만들 필요가 없어서 편하다.

### 방법 B: Claude Code로 만들기

이미 빈 폴더가 있다면, 해당 폴더에서 Claude Code를 열고 아래 1.4절의 구현 프롬프트를 사용하면 된다. 생성 후 IntelliJ에서 `File` → `Open` → 해당 폴더를 선택하면 프로젝트가 열린다.

## 1.3 CLAUDE.md 만들기

> **CLAUDE.md란?** Claude Code가 대화를 시작할 때 자동으로 읽는 "프로젝트 설명서"다. 여기에 "이 프로젝트는 Java 17이고, 이런 규칙을 따라"라고 적어두면, 매번 말하지 않아도 Claude Code가 알아서 맥락을 이해한다. `.gitignore`처럼 프로젝트 폴더 최상단에 놓는 설정 파일이라고 생각하면 된다.

### 왜 필요한가

- **매번 설명 안 해도 됨:** "이 프로젝트는 Java 17 + Spring Boot 3.3.x야"를 매 대화마다 말할 필요가 없다
- **일관된 코드:** 패키지 구조, 네이밍 규칙 등을 고정해두면 코드가 일관되게 나온다
- **실수 방지:** "아직 Security 의존성은 추가하지 마" 같은 제약을 적어두면 앞서나가는 실수를 막을 수 있다

### CLAUDE.md 생성 프롬프트

Claude Code에 아래를 입력한다:

```
프로젝트 루트에 CLAUDE.md 파일을 생성해줘. 아래 내용을 포함해야 해:

1. 프로젝트 개요
   - 프로젝트명: TaskSyncBoard (사내 협업/이슈 관리 시스템, Jira 클론)
   - 학습 목적 프로젝트이며, TASKSYNCBOARD_STEP_BY_STEP.md를 따라 단계별로 구현 중

2. 기술 스택 (현재 Phase 1 기준)
   - Java 17
   - Spring Boot 3.3.x
   - Gradle (Groovy DSL)
   - 현재 의존성: Spring Web, Lombok, spring-boot-starter-test (이 외 추가 금지)

3. 프로젝트 구조 규칙
   - 베이스 패키지: com.tasksyncboard
   - 도메인별 패키지: com.tasksyncboard.domain.{도메인명}.{entity|repository|service|controller|dto}
   - 공통 패키지: com.tasksyncboard.global.{config|error|common|security|aop|logging}

4. 코딩 컨벤션
   - Lombok 사용: @Getter, @NoArgsConstructor(access = PROTECTED), @Builder, @RequiredArgsConstructor
   - 필드 주입(@Autowired) 금지, 생성자 주입만 사용
   - DTO와 Entity를 반드시 분리
   - 클래스/메서드에 한글 주석으로 역할 설명 포함

5. 현재 진행 상태
   - Phase 1: 프로젝트 초기 세팅 (진행 중)
   - 아직 추가하면 안 되는 것: JPA, Spring Security, Redis, QueryDSL 등 (해당 Phase에서 추가 예정)
   
6. Claude Code 협업 규칙
   - git commit/push는 사용자가 직접 처리한다. Claude는 파일 수정만 하고, `git add / commit / push` 명령을 자동으로 실행하지 않는다.

7. 빌드/실행 명령어
   - 빌드: ./gradlew build
   - 실행: ./gradlew bootRun
   - 테스트: ./gradlew test
```

> **팁:** Phase가 진행될 때마다 CLAUDE.md의 "기술 스택"과 "현재 진행 상태"를 업데이트하자. 예를 들어 Phase 3에서 JPA를 추가하면 기술 스택에 JPA를 넣고, "추가하면 안 되는 것"에서 JPA를 빼면 된다.

## 1.4 프로젝트 구조 잡기 (Claude Code 프롬프트)

### 구현 프롬프트

IntelliJ에서 프로젝트를 직접 만들었다면(방법 A), 기본 파일들이 이미 생성되어 있다. Claude Code에 아래 프롬프트를 입력해서 나머지 구조를 완성하자.

```
TaskSyncBoard 프로젝트의 초기 세팅을 해줘. 아래 요구사항을 정확히 따라줘.

1. build.gradle에 의존성이 Spring Web + Lombok만 있는지 확인해줘.
   (JPA, Security, Redis 등은 나중에 추가할 예정이니 지금은 넣지 마.)

2. 패키지 구조를 생성해줘:
   - com.tasksyncboard.global.config
   - com.tasksyncboard.global.error
   - com.tasksyncboard.global.common
   - com.tasksyncboard.global.security
   - com.tasksyncboard.global.aop
   - com.tasksyncboard.global.logging
   - com.tasksyncboard.domain.member.entity
   - com.tasksyncboard.domain.member.repository
   - com.tasksyncboard.domain.member.service
   - com.tasksyncboard.domain.member.controller
   - com.tasksyncboard.domain.member.dto
   - com.tasksyncboard.domain.project (같은 하위 구조)
   - com.tasksyncboard.domain.issue (같은 하위 구조)
   - com.tasksyncboard.domain.comment (같은 하위 구조)

3. application.yml을 생성해줘:
   - spring.profiles.active: local
   - spring.application.name: task-sync-board

4. application-local.yml을 생성해줘 (지금은 서버 포트 설정만):
   - server.port: 8080

5. .gitignore를 생성해줘 (Gradle, IDE, OS 파일 제외)

6. TaskSyncBoardApplication.java 메인 클래스가 없으면 생성해줘.
```

> 만약 프로젝트를 처음부터 Claude Code로 만들 거라면(방법 B), 위 프롬프트 맨 앞에 아래를 추가한다:
> ```
> 0. Spring Initializr 기반으로 Gradle(Groovy) 프로젝트를 생성해줘:
>    - Group: com.tasksyncboard
>    - Artifact: task-sync-board
>    - Java 17
>    - Spring Boot 3.3.x (최신 안정 버전)
>    - Dependencies: Spring Web, Lombok (이 두 개만!)
> ```

## 1.5 실행해보기

### IntelliJ에서 실행하는 법

1. IntelliJ 왼쪽 프로젝트 탐색기에서 `src/main/java/com/tasksyncboard/TaskSyncBoardApplication.java` 파일을 연다
2. `main` 메서드 왼쪽의 **초록색 재생 버튼(▶)** 클릭 → `Run` 선택
3. 아래쪽 콘솔 창에 이런 메시지가 나오면 성공:
   ```
   Started TaskSyncBoardApplication in X.XX seconds
   ```
4. 웹 브라우저를 열고 주소창에 `http://localhost:8080` 입력
   - **"Whitelabel Error Page"가 나오면 정상이다!** 아직 페이지를 안 만들었으니 에러 페이지가 나오는 게 맞다. 서버가 돌아가고 있다는 뜻이다.
5. 서버 끄기: IntelliJ 하단 콘솔의 **빨간색 네모 버튼(■)** 클릭

### 터미널에서 실행하는 법

IntelliJ 하단의 `Terminal` 탭을 클릭하거나, 별도 터미널을 열고:

```bash
./gradlew bootRun
```

서버를 끄려면 `Ctrl + C`를 누른다.

### 확인 프롬프트 (Claude Code용)

```
Phase 1 세팅이 완료되었는지 확인해줘:

1. build.gradle을 읽고 의존성이 Spring Web + Lombok만 있는지 확인해줘.
2. 패키지 구조가 설계대로 생성되었는지 확인해줘.
3. './gradlew build' 명령으로 빌드가 성공하는지 확인해줘.
4. './gradlew bootRun'으로 애플리케이션이 8080 포트에서 정상 시작되는지 확인해줘.
5. 빌드 에러가 있다면 원인을 분석하고 수정해줘.
```

---

여기까지 하면 프로젝트가 실행되는 것을 확인했을 것이다. 축하한다! 아래부터는 **"우리가 방금 만든 것"에 대한 설명**이다. 지금 전부 이해 못 해도 전혀 괜찮다. Phase를 진행하면서 자연스럽게 익숙해진다. 지금 당장은 넘어가고 나중에 돌아와서 읽어도 된다.

---

## 1.6 우리가 방금 만든 것 이해하기

### Spring Boot가 뭔데?

한 줄 요약: **Java로 웹 서비스를 쉽게 만들게 해주는 도구(프레임워크)**다.

원래 Java로 웹 서버를 만들려면 설정할 게 엄청 많았다. Spring Boot는 이 설정을 거의 자동으로 해준다. 우리가 한 일을 생각해보자:

- 의존성 2개만 선택했다 (Spring Web, Lombok)
- `main` 메서드 하나만 실행했다
- 그런데 웹 서버가 돌아간다!

이게 가능한 이유는 Spring Boot가 "이 프로젝트에 Spring Web이 있네? 그러면 웹 서버(Tomcat)를 알아서 켜줄게"라고 자동으로 처리해주기 때문이다. 이걸 **자동 설정(Auto-Configuration)**이라고 한다.

### build.gradle은 뭘 하는 파일이야?

**"이 프로젝트에 필요한 재료 목록"**이라고 생각하면 된다.

요리에 비유하면:
- `build.gradle` = 레시피 (필요한 재료 + 만드는 방법)
- `dependencies` 블록 = 재료 목록 (Spring Web, Lombok 등)
- `Gradle` = 재료를 인터넷에서 자동으로 가져다주는 배달 서비스

```groovy
dependencies {
    // "웹 서버 기능이 필요해" → Gradle이 알아서 가져옴
    implementation 'org.springframework.boot:spring-boot-starter-web'

    // "Lombok이 필요해 (코드 작성할 때만)" → 컴파일할 때만 사용
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // "테스트 도구가 필요해 (테스트할 때만)" → 테스트 코드에서만 사용
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

> 지금은 이 3개가 전부다. 나중에 Phase가 진행되면서 JPA, Security, Redis 등을 여기에 하나씩 추가해 나간다.

### 패키지 구조는 왜 이렇게 나눴어?

```
com.tasksyncboard/
├── global/          ← 프로젝트 전체에서 공통으로 쓰는 것
│   ├── config/      ← 설정 모음
│   ├── error/       ← 에러 처리
│   ├── common/      ← 공통 코드
│   └── ...
└── domain/          ← 실제 비즈니스 기능별로 분류
    ├── member/      ← 회원 관련 코드 전부 여기
    │   ├── entity/      ← 데이터 구조 (DB 테이블과 대응)
    │   ├── repository/  ← DB에서 데이터 꺼내기/저장하기
    │   ├── service/     ← 비즈니스 로직 (핵심 기능)
    │   ├── controller/  ← 외부 요청 받는 입구 (API)
    │   └── dto/         ← 데이터를 주고받을 때 쓰는 그릇
    ├── project/     ← 프로젝트(워크스페이스) 관련
    ├── issue/       ← 이슈(티켓) 관련
    └── comment/     ← 댓글 관련
```

이런 구조를 **"도메인 중심 구조"**라고 한다. 회원에 관한 코드는 `member/` 폴더에, 이슈에 관한 코드는 `issue/` 폴더에 모아두는 방식이다.

왜 이렇게 할까? 나중에 "이슈 기능을 수정해야지" 하면 `issue/` 폴더만 보면 되니까 코드를 찾기 쉽다. 비유하자면, 옷장을 상의/하의/속옷으로 나누는 것과 같다.

### @SpringBootApplication은 뭘 하는 거야?

```java
@SpringBootApplication
public class TaskSyncBoardApplication {
    public static void main(String[] args) {
        SpringApplication.run(TaskSyncBoardApplication.class, args);
    }
}
```

이 파일이 우리 프로젝트의 **시작 버튼**이다. `SpringApplication.run()`을 실행하면 Spring Boot가:

1. 프로젝트에 어떤 라이브러리가 있는지 확인한다
2. 필요한 설정을 자동으로 해준다
3. 우리가 만든 코드들을 찾아서 등록한다
4. 웹 서버(Tomcat)를 켠다

> 지금은 "이게 시작 버튼이구나" 정도만 알면 충분하다.

### application.yml은 뭔데?

프로젝트의 **설정 파일**이다. "어떤 포트에서 서버를 열지", "어떤 데이터베이스를 쓸지" 같은 정보를 여기에 적는다.

```yaml
# application.yml - 공통 설정
spring:
  profiles:
    active: local              # "local 환경으로 실행할게"
  application:
    name: task-sync-board      # 프로젝트 이름
```

```yaml
# application-local.yml - 내 컴퓨터(로컬) 전용 설정
server:
  port: 8080                   # 8080번 포트에서 서버 열기
```

왜 파일을 나누나? 나중에 실제 서버에 올릴 때는 `application-prod.yml`을 따로 만들어서 다른 설정(다른 DB 주소, 다른 포트 등)을 쓸 수 있다. 환경별로 설정을 분리해두는 것이다.

## 1.7 [더 알아보기] IoC와 DI

> **이 섹션은 건너뛰어도 된다.** Phase 4에서 실제로 Service, Controller 코드를 만들 때 자연스럽게 체감하게 된다. 지금 이해가 안 되면 넘어가고, 나중에 다시 와서 읽어도 된다.

### IoC(제어의 역전)를 쉽게 설명하면

식당에 비유해보자:

- **IoC 없이:** 손님이 직접 주방에 가서 재료를 꺼내고, 요리하고, 접시에 담는다
- **IoC 있으면:** 손님은 메뉴만 고르면 된다. 식당(Spring)이 요리사를 배정하고, 재료를 준비하고, 음식을 가져다준다

코드로 보면:

```java
// IoC 없이: 개발자가 직접 만든다
public class IssueService {
    private IssueRepository repo = new IssueRepository();  // 직접 생성
}

// IoC 있음: Spring이 알아서 넣어준다
@Service
public class IssueService {
    private final IssueRepository repo;  // Spring이 자동으로 넣어줌

    public IssueService(IssueRepository repo) {
        this.repo = repo;
    }
}
```

"제어가 역전됐다"는 말은, **객체를 만들고 연결하는 일**을 개발자가 아니라 Spring이 대신 해준다는 뜻이다.

### DI(의존성 주입)는 뭔데?

IoC를 실현하는 구체적인 방법이다. Spring이 필요한 객체를 **자동으로 넣어주는(주입하는) 것**을 DI라고 한다.

DI를 하는 방법은 3가지가 있는데, 이 프로젝트에서는 **생성자 주입**만 사용한다:

```java
@Service
@RequiredArgsConstructor  // Lombok이 생성자를 자동으로 만들어줌
public class IssueService {
    private final IssueRepository issueRepository;  // Spring이 여기에 알아서 넣어줌
}
```

왜 생성자 주입을 쓰나?
- `final`을 붙일 수 있어서 값이 중간에 바뀌지 않는다 (안전)
- 테스트할 때 가짜 객체를 넣기 쉽다
- 필요한 게 빠지면 시작할 때 바로 에러가 나서 문제를 빨리 찾을 수 있다

> **`@RequiredArgsConstructor`가 뭐야?** Lombok이 제공하는 기능이다. `final`이 붙은 변수들을 받는 생성자를 자동으로 만들어준다. 이게 없으면 아래처럼 직접 써야 한다:
> ```java
> public IssueService(IssueRepository issueRepository) {
>     this.issueRepository = issueRepository;
> }
> ```

### 다른 DI 방식은 왜 안 쓰나? (참고)

```java
// 1. 필드 주입 - 쓰지 않는다
@Autowired
private IssueRepository issueRepository;
// → final 불가, 테스트 어려움, 문제를 늦게 발견

// 2. 세터 주입 - 거의 안 씀
@Autowired
public void setIssueRepository(IssueRepository repo) { ... }
// → 주입 전에 메서드가 호출될 위험
```

## 1.8 [더 알아보기] Bean과 생명주기

> **이 섹션도 건너뛰어도 된다.** Phase가 진행되면서 필요한 순간이 오면 돌아와서 읽으면 된다.

### Bean이 뭔데?

Spring이 만들고 관리하는 객체를 **Bean(빈)**이라고 부른다. 우리가 `@Service`, `@Controller`, `@Repository` 같은 표시(어노테이션)를 클래스에 붙이면, Spring이 "아, 이건 내가 관리해야 하는 객체구나"라고 인식하고 자동으로 만들어준다.

```java
@Service     // ← "이건 Spring이 관리하는 Bean이야"
public class IssueService { ... }

@Controller  // ← "이것도 Bean이야"
public class IssueController { ... }
```

### Bean이 만들어지는 과정 (간략 버전)

```
1. Spring 시작
      ↓
2. @Service, @Controller 등이 붙은 클래스를 찾는다 (Component Scan)
      ↓
3. 찾은 클래스들의 객체(Bean)를 만든다
      ↓
4. Bean끼리 필요한 것을 연결해준다 (DI)
      ↓
5. 초기화 작업 (@PostConstruct 등)
      ↓
6. 사용 가능! (요청을 받을 준비 완료)
      ↓
7. Spring 종료 시 정리 작업 (@PreDestroy 등)
```

### 하나만 기억하자: Singleton

우리 프로젝트의 모든 Bean은 **딱 하나만** 만들어져서 모든 요청이 같은 객체를 공유한다. 이걸 Singleton(싱글턴)이라고 한다.

이게 왜 중요하냐면, 하나의 객체를 여러 요청이 동시에 쓰니까 **Bean 안에 데이터를 저장하면 안 된다:**

```java
@Service
public class IssueService {
    private int count = 0;  // 이러면 안 됨! 여러 요청이 동시에 바꿔서 엉킴
    private final IssueRepository repo;  // 이건 OK. 다른 Bean을 가리킬 뿐
}
```

> **핵심 한 줄:** "Bean은 Spring이 대신 만들고 관리해주는 객체다." 지금은 이것만 기억하면 된다.

---

# Phase 2: Docker Compose 개발 환경 (PostgreSQL + Redis)

## 이 Phase에서 추가하는 의존성

**없음.** 이 Phase에서는 build.gradle을 수정하지 않는다. Docker 컨테이너만 준비한다. 애플리케이션에서 DB와 Redis에 실제로 접속하는 것은 Phase 3(JPA)과 Phase 5(Redis)에서 한다.

## 2.1 왜 Docker Compose인가

로컬 개발 환경에서 PostgreSQL과 Redis를 직접 설치하면:
- OS마다 설치 방법이 다르다
- 버전 관리가 어렵다
- 팀원 간 환경이 달라 "내 PC에서는 되는데" 문제가 발생한다
- 프로젝트별로 다른 버전이 필요할 수 있다

**Docker Compose는 `docker-compose.yml` 하나로 모든 인프라를 정의하고, `docker-compose up` 한 줄로 동일한 환경을 띄운다.** 이것이 "Infrastructure as Code"의 첫걸음이다.

## 2.2 docker-compose.yml 설계

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: tsb-postgres
    environment:
      POSTGRES_DB: tasksyncboard
      POSTGRES_USER: tsb_user
      POSTGRES_PASSWORD: tsb_password
    ports:
      - "5432:5432"
    volumes:
      - tsb-postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tsb_user -d tasksyncboard"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: tsb-redis
    ports:
      - "6379:6379"
    volumes:
      - tsb-redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  tsb-postgres-data:
  tsb-redis-data:
```

### 설정 상세 설명

- **`image: postgres:16-alpine`**: Alpine Linux 기반의 경량 이미지. 프로덕션에서도 PostgreSQL 16을 사용한다면 개발 환경도 동일 버전을 맞추는 것이 좋다.
- **`volumes`**: Named Volume을 사용하여 컨테이너가 재시작되어도 데이터가 유지된다. `docker-compose down`으로 중지 후 다시 `up`해도 데이터가 살아있다. 완전 초기화하려면 `docker-compose down -v` (볼륨도 삭제).
- **`healthcheck`**: 컨테이너가 "시작됨(started)"과 "준비됨(healthy)"을 구분한다. PostgreSQL 컨테이너가 시작되어도 실제로 접속 가능하기까지 몇 초 걸릴 수 있다. healthcheck가 있으면 다른 서비스가 `depends_on: condition: service_healthy`로 "준비될 때까지" 기다릴 수 있다.

> **Redis는 Phase 5(JWT + Redis 인증)에서 처음 사용하지만, 여기서 미리 Docker Compose에 포함시킨다.** 인프라는 한 곳에서 관리하는 것이 편하기 때문이다.

## 2.3 Phase 2 Claude Code 구현 프롬프트

### 구현 프롬프트

```
TaskSyncBoard 프로젝트에 Docker Compose 개발 환경을 구성해줘.

1. docker/docker-compose.yml 파일을 생성해줘:
   - PostgreSQL 16 Alpine (포트 5432, DB: tasksyncboard, 사용자: tsb_user, 비밀번호: tsb_password)
   - Redis 7 Alpine (포트 6379)
   - 둘 다 Named Volume과 healthcheck 설정 포함

2. build.gradle은 수정하지 마. 아직 JPA나 Redis 의존성을 추가하지 않는다.
   (다음 Phase에서 JPA를 추가할 때 DB에 접속할 예정)
```

### 확인 및 테스트 프롬프트

```
Docker Compose 환경이 올바르게 설정되었는지 확인해줘:

1. docker-compose.yml 파일 내용을 확인해줘.
2. 'docker-compose -f docker/docker-compose.yml up -d' 명령으로 컨테이너를 시작해줘.
3. 'docker-compose -f docker/docker-compose.yml ps'로 컨테이너 상태를 확인해줘 (둘 다 healthy인지).
4. PostgreSQL 접속 테스트: 'docker exec tsb-postgres psql -U tsb_user -d tasksyncboard -c "SELECT 1"'
5. Redis 접속 테스트: 'docker exec tsb-redis redis-cli ping'
```

---

# Phase 3: 도메인 모델링 + JPA 딥다이브 + Flyway

## 이 Phase에서 추가하는 의존성

| 라이브러리 | 용도 |
|-----------|------|
| `spring-boot-starter-data-jpa` | Spring Data JPA + Hibernate (ORM) |
| `postgresql` | PostgreSQL JDBC 드라이버 |
| `flyway-core` | DB 스키마 버전 관리 (마이그레이션 도구) |
| `flyway-database-postgresql` | Flyway PostgreSQL 지원 모듈 |

### build.gradle에 추가할 내용

```groovy
dependencies {
    // === Phase 1: 프로젝트 뼈대 === //
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    // === Phase 3: JPA + DB === //
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql'
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql'
}
```

> **왜 이제 JPA를 추가하는가?** Phase 2에서 Docker로 PostgreSQL을 띄웠으니, 이제 JPA로 DB에 접속하고 엔티티를 매핑할 수 있다. JPA 없이는 도메인 모델링이 불가능하다.

### application-local.yml에 추가할 설정

Phase 1에서 서버 포트만 있던 `application-local.yml`에 DB 연결 설정을 추가한다:

```yaml
server:
  port: 8080

# === Phase 3에서 추가 === #
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/tasksyncboard
    username: tsb_user
    password: tsb_password
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 100
    open-in-view: false

  flyway:
    enabled: true
    locations: classpath:db/migration

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

## 3.1 JPA(Java Persistence API) 딥다이브

> **이 섹션이 Phase 3에 있는 이유:** JPA를 지금 처음 사용하므로, JPA가 무엇인지 여기서 설명한다.

### JPA vs Hibernate vs Spring Data JPA의 관계

이 세 가지의 관계를 명확히 이해해야 한다:

```
┌──────────────────────────────────────────────────────┐
│  Spring Data JPA                                     │
│  ┌────────────────────────────────────────────────┐  │
│  │  JPA (Jakarta Persistence API) - 표준 스펙      │  │
│  │  ┌──────────────────────────────────────────┐  │  │
│  │  │  Hibernate - JPA 구현체                   │  │  │
│  │  │  (EclipseLink, OpenJPA 등도 구현체)       │  │  │
│  │  └──────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

- **JPA:** Java 표준 ORM 스펙(인터페이스 모음). `@Entity`, `@Id`, `@ManyToOne` 등의 어노테이션을 정의한다. 그 자체로는 동작하지 않는다.
- **Hibernate:** JPA 스펙의 **구현체**. 실제로 SQL을 생성하고 실행하며, 영속성 컨텍스트를 관리한다. Spring Boot의 기본 JPA Provider이다.
- **Spring Data JPA:** JPA/Hibernate 위에 추상화 계층을 얹어, `JpaRepository` 인터페이스만 정의하면 **CRUD 구현체를 자동 생성**해준다.

### 영속성 컨텍스트 (Persistence Context) 딥다이브

**영속성 컨텍스트는 JPA의 핵심 중의 핵심이다.** 이것을 이해하지 못하면 JPA에서 겪는 대부분의 버그를 디버깅할 수 없다.

영속성 컨텍스트란 "**엔티티를 영구 저장하는 환경**"이다. 논리적인 개념으로, `EntityManager`를 통해 접근한다. Spring에서는 **트랜잭션 범위 = 영속성 컨텍스트 범위**이다.

#### 엔티티의 4가지 상태

```
                    persist()
  [비영속 New] ──────────────► [영속 Managed]
       │                           │ │
       │                   find()  │ │  flush() / commit()
       │                     ◄─────┘ │──────► DB에 SQL 전송
       │                             │
       │                    remove() │
       │              ┌─────────────►│
       │              │              ▼
       │         [삭제 Removed]  [준영속 Detached]
       │                             ▲
       │                             │ detach() / clear() / close()
       │                             │ 또는 트랜잭션 종료
       └─────────────────────────────┘
```

**1. 비영속(New/Transient):** `new Member()`로 생성만 한 상태. 영속성 컨텍스트와 무관.

**2. 영속(Managed):** `em.persist(member)` 또는 `em.find()`로 영속성 컨텍스트에 등록된 상태.

**3. 준영속(Detached):** 영속성 컨텍스트에서 분리된 상태. 트랜잭션이 끝나면 엔티티는 준영속이 된다.

**4. 삭제(Removed):** `em.remove()`로 삭제 예정으로 표시된 상태.

#### 영속성 컨텍스트의 핵심 기능

**1차 캐시 (First-Level Cache):**
```java
@Transactional
public void example() {
    // DB에서 조회 → 1차 캐시에 저장 → SQL 1회 실행
    Member member1 = memberRepository.findById(1L).get();

    // 1차 캐시에서 조회 → SQL 실행 안 함!
    Member member2 = memberRepository.findById(1L).get();

    // 같은 객체인가? → YES! (같은 트랜잭션 안에서는 동일성 보장)
    assert member1 == member2; // true
}
```

1차 캐시는 **트랜잭션 범위**로 동작한다. 트랜잭션이 끝나면 1차 캐시도 사라진다. 이것은 애플리케이션 레벨의 캐시(2차 캐시, Redis 등)와는 다르다.

**Dirty Checking (변경 감지):**
```java
@Transactional
public void updateMemberName(Long id, String newName) {
    Member member = memberRepository.findById(id).get(); // 영속 상태
    member.setName(newName); // setter만 호출! save() 필요 없음!

    // 트랜잭션 커밋 시점에:
    // 1. 영속성 컨텍스트가 "처음 읽었을 때의 스냅샷"과 "현재 상태"를 비교
    // 2. 변경이 감지되면 UPDATE SQL 자동 생성
    // 3. flush → DB에 UPDATE 전송 → commit
}
```

이것이 Dirty Checking이다. **영속 상태의 엔티티를 수정하면, 트랜잭션 커밋 시점에 변경을 감지하고 자동으로 UPDATE SQL을 실행한다.** `save()`를 다시 호출할 필요가 없다.

**Write-Behind (쓰기 지연):**
```java
@Transactional
public void createMembers() {
    memberRepository.save(member1); // INSERT SQL을 즉시 실행하지 않음! 쓰기 지연 SQL 저장소에 적재
    memberRepository.save(member2); // 마찬가지로 적재만
    memberRepository.save(member3); // 적재만

    // 트랜잭션 커밋 시점에 한꺼번에 flush!
    // → INSERT 3개가 DB에 전송됨
}
```

### 지연 로딩 (Lazy Loading) vs 즉시 로딩 (Eager Loading)

```java
@Entity
public class Issue {
    @ManyToOne(fetch = FetchType.LAZY)  // 지연 로딩
    private Member assignee;

    @ManyToOne(fetch = FetchType.EAGER) // 즉시 로딩
    private Member reporter;
}
```

**즉시 로딩(EAGER):** Issue를 조회하면 reporter도 **무조건 함께** JOIN하여 가져온다. 편리해 보이지만 **위험하다:**
- 연관된 엔티티가 또 다른 EAGER 연관을 가지면 연쇄적으로 JOIN이 발생
- 필요 없는 데이터도 항상 가져와서 성능 저하
- 예측 불가능한 SQL이 발생하여 디버깅이 어려움

**지연 로딩(LAZY):** Issue를 조회하면 assignee는 **프록시 객체**로 채워진다. 실제 assignee의 필드에 접근하는 순간에 SELECT SQL이 실행된다.

```java
Issue issue = issueRepository.findById(1L).get();
// 이 시점에는 assignee가 프록시 객체 (DB 조회 안 함)

issue.getAssignee().getName(); // ← 이 순간! SELECT * FROM member WHERE id = ? 실행
```

> **실무 원칙: 모든 연관관계는 LAZY로 설정하고, 필요할 때만 fetch join으로 한꺼번에 가져온다.** `@ManyToOne`의 기본값은 EAGER이므로 반드시 `fetch = FetchType.LAZY`를 명시해야 한다.

#### 프록시 객체란?

Hibernate는 지연 로딩을 위해 **프록시 패턴**을 사용한다:

```
  ┌─────────────────────────────────────┐
  │  MemberProxy (Hibernate가 생성)      │
  │  extends Member                      │
  │                                      │
  │  - target: Member = null (아직 미로드) │
  │  - loaded: false                     │
  │                                      │
  │  + getName() {                       │
  │      if (!loaded) {                  │
  │          target = DB에서 조회;         │
  │          loaded = true;              │
  │      }                               │
  │      return target.getName();        │
  │  }                                   │
  └─────────────────────────────────────┘
```

Hibernate는 **바이트코드 조작(ByteBuddy/Javassist)**으로 엔티티 클래스를 상속한 프록시 클래스를 런타임에 생성한다. 이 프록시는 실제 엔티티처럼 행동하지만, 필드 접근 시점에 DB를 조회한다.

**주의: LazyInitializationException**
```java
public Issue findIssue(Long id) {
    Issue issue = issueRepository.findById(id).get(); // 트랜잭션 안
    return issue;
} // 트랜잭션 종료 → 영속성 컨텍스트 종료

// Controller에서...
issue.getAssignee().getName(); // LazyInitializationException!
// 영속성 컨텍스트가 이미 닫혀서 프록시가 DB를 조회할 수 없음
```

해결 방법:
1. **fetch join** (권장): `@Query("SELECT i FROM Issue i JOIN FETCH i.assignee WHERE i.id = :id")`
2. **@EntityGraph**: `@EntityGraph(attributePaths = {"assignee"})`
3. **DTO 프로젝션** (가장 권장): 서비스 레이어에서 필요한 데이터만 DTO로 변환하여 반환

### N+1 문제 (Phase 7에서 상세히 다루지만 개념은 여기서)

```java
// 모든 이슈를 조회
List<Issue> issues = issueRepository.findAll(); // SQL 1번: SELECT * FROM issue

for (Issue issue : issues) {
    // 각 이슈의 assignee에 접근할 때마다 SQL 1번씩 추가 발생!
    System.out.println(issue.getAssignee().getName());
    // SQL N번: SELECT * FROM member WHERE id = ?
}
// 총 1 + N 번의 SQL 실행! (이슈가 100개면 101번)
```

> Phase 7에서 QueryDSL과 함께 N+1 문제의 해결 방법(Fetch Join, @BatchSize, DTO Projection)을 상세히 다룬다.

## 3.2 트랜잭션 전파 속성 (Transaction Propagation) 딥다이브

`@Transactional`의 `propagation` 속성은 **이미 트랜잭션이 진행 중일 때, 새로운 트랜잭션을 어떻게 처리할 것인가**를 결정한다.

```java
@Service
public class IssueService {
    @Transactional // 트랜잭션 A 시작
    public void createIssue(IssueCreateRequest request) {
        Issue issue = Issue.create(request);
        issueRepository.save(issue);

        historyService.recordHistory(issue); // ← 이 메서드에도 @Transactional이 있다면?
    }
}

@Service
public class HistoryService {
    @Transactional(propagation = Propagation.???) // 어떤 전파 속성을 쓸 것인가?
    public void recordHistory(Issue issue) {
        historyRepository.save(new IssueHistory(issue));
    }
}
```

| 전파 속성 | 동작 | 실무 사용 시점 |
|-----------|------|---------------|
| `REQUIRED` (기본) | 기존 트랜잭션이 있으면 참여, 없으면 새로 생성 | 대부분의 서비스 메서드 |
| `REQUIRES_NEW` | **항상** 새 트랜잭션 생성 (기존 것은 일시 중단) | 독립적인 작업 (로깅, 알림 등 메인 로직 실패와 무관하게 기록해야 할 때) |
| `SUPPORTS` | 기존 트랜잭션이 있으면 참여, 없으면 트랜잭션 없이 실행 | 조회 전용 메서드 |
| `MANDATORY` | 기존 트랜잭션이 반드시 있어야 함. 없으면 예외 | 반드시 상위 트랜잭션 내에서만 호출되어야 하는 메서드 |
| `NOT_SUPPORTED` | 트랜잭션 없이 실행 (기존 것은 일시 중단) | 트랜잭션이 불필요한 외부 API 호출 등 |
| `NEVER` | 트랜잭션이 있으면 예외 | 트랜잭션 컨텍스트에서 호출되면 안 되는 메서드 |
| `NESTED` | 기존 트랜잭션 내에 중첩 트랜잭션(Savepoint) 생성 | 부분 롤백이 필요한 경우 (JPA에서는 지원 제한적) |

#### 핵심: @Transactional과 프록시

**@Transactional은 AOP로 동작한다!** 즉, **프록시를 통해서만** 트랜잭션이 적용된다. 이 말은:

```java
@Service
public class IssueService {
    @Transactional
    public void methodA() {
        this.methodB(); // ← 트랜잭션이 적용되지 않음!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() {
        // 같은 클래스 내부에서 this로 호출하면 프록시를 거치지 않으므로
        // REQUIRES_NEW가 무시되고, methodA의 트랜잭션에 참여하게 됨
    }
}
```

**같은 클래스 내의 메서드를 `this`로 호출하면 프록시를 거치지 않는다!** 이것은 Spring AOP의 가장 흔한 함정이다. 해결 방법:
1. `methodB()`를 별도의 클래스(서비스)로 분리
2. `self` 주입 패턴 (비권장)
3. `ApplicationContext.getBean()` (비권장)

## 3.3 Flyway - DB 스키마 형상 관리

### Flyway란?

애플리케이션 코드는 Git으로 버전 관리하면서, **DB 스키마는 왜 버전 관리하지 않는가?**

개발 중에 `ALTER TABLE` 스크립트를 수동으로 실행하다 보면:
- 누가 어떤 변경을 했는지 추적 불가
- 개발/스테이징/프로덕션 DB 스키마가 서로 달라짐
- 팀원이 합류했을 때 DB 초기 세팅이 어려움
- 롤백이 불가능

**Flyway는 DB 스키마를 코드(SQL 파일)로 관리하고, 애플리케이션 시작 시 자동으로 마이그레이션한다.**

### 동작 원리

```
src/main/resources/db/migration/
├── V1__create_member_table.sql        ← 버전 1
├── V2__create_project_table.sql       ← 버전 2
├── V3__create_issue_table.sql         ← 버전 3
└── V4__add_priority_to_issue.sql      ← 버전 4
```

**파일 이름 규칙:** `V{버전번호}__{설명}.sql` (V 다음에 숫자, **언더스코어 두 개**, 설명)

**동작 흐름:**
1. 애플리케이션 시작 시 Flyway가 동작
2. DB에 `flyway_schema_history` 테이블이 있는지 확인 (없으면 생성)
3. 이 테이블에서 "현재까지 적용된 마지막 버전"을 확인
4. `db/migration/` 폴더에서 아직 적용되지 않은 SQL 파일을 순서대로 실행
5. 실행 결과를 `flyway_schema_history`에 기록

**중요한 규칙:**
- **한번 적용된 마이그레이션 파일은 절대 수정하지 않는다!** (Flyway가 체크섬으로 검증)
- 스키마를 변경하려면 **새로운 버전의 마이그레이션 파일을 추가**한다
- 이것은 Git commit과 같은 개념: 기존 커밋을 수정하지 않고, 새 커밋을 추가하는 것

### 왜 ddl-auto: validate를 쓰는가?

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # create, update가 아닌 validate!
```

| 옵션 | 동작 | 사용 시점 |
|------|------|-----------|
| `create` | 매번 테이블 DROP 후 CREATE | 절대 쓰지 마라 (데이터 소실) |
| `create-drop` | 시작 시 CREATE, 종료 시 DROP | 테스트 전용 |
| `update` | 엔티티와 테이블 비교 후 ALTER | 위험! 컬럼 삭제 안 됨, 인덱스 관리 불가 |
| `validate` | 엔티티와 테이블 일치 여부만 검증 | **프로덕션 + Flyway 조합 시 필수** |
| `none` | 아무것도 안 함 | - |

**Flyway로 스키마를 관리하면, `ddl-auto: validate`를 써서 "Flyway가 만든 테이블"과 "JPA 엔티티"가 일치하는지 검증만 한다.** 만약 불일치하면 애플리케이션이 시작 시점에 에러를 내므로, 배포 전에 문제를 잡을 수 있다.

## 3.4 BaseEntity (공통 Auditing)

모든 테이블에 `created_at`, `updated_at`, `created_by`, `updated_by`를 넣으려면 매번 중복 코드를 작성해야 한다. **JPA Auditing**으로 이를 자동화한다.

```java
@Getter
@MappedSuperclass // ← 이 클래스는 테이블로 매핑되지 않고, 상속받는 엔티티에 필드만 제공
@EntityListeners(AuditingEntityListener.class) // ← Auditing 이벤트 리스너 등록
public abstract class BaseEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;

    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;
}
```

`@MappedSuperclass`의 역할: 이 어노테이션이 붙은 클래스는 **JPA 엔티티가 아니다**. 테이블에 매핑되지 않는다. 하지만 이 클래스를 상속받는 엔티티는 이 클래스의 필드를 **자기 테이블의 컬럼으로** 가진다.

`@CreatedBy` / `@LastModifiedBy`가 동작하려면 `AuditorAware<String>` 구현체를 Bean으로 등록해야 한다:

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> {
            // Phase 5에서 Spring Security를 추가하면 SecurityContext에서 사용자를 가져온다.
            // 지금은 임시로 "SYSTEM"을 반환한다.
            return Optional.of("SYSTEM");
        };
    }
}
```

> **AuditorAware는 Phase 5에서 Security를 추가한 후 실제 로그인 사용자 정보로 교체한다.** 지금은 "SYSTEM"으로 임시 처리한다.

## 3.5 엔티티 설계

### Member 엔티티

```java
@Entity
@Table(name = "members")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_id")
    private Long id;

    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @Column(nullable = false)
    private String password;

    @Column(nullable = false, length = 50)
    private String name;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private MemberRole role; // ADMIN, MANAGER, MEMBER

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private MemberStatus status; // ACTIVE, INACTIVE, WITHDRAWN

    // === 생성 메서드 (정적 팩토리 메서드 패턴) === //
    public static Member create(String email, String encodedPassword, String name) {
        Member member = new Member();
        member.email = email;
        member.password = encodedPassword;
        member.name = name;
        member.role = MemberRole.MEMBER;
        member.status = MemberStatus.ACTIVE;
        return member;
    }
}
```

#### 왜 이렇게 설계하는가?

**1. `@NoArgsConstructor(access = AccessLevel.PROTECTED)`:**
JPA 스펙상 엔티티는 기본 생성자가 필요하다 (리플렉션으로 인스턴스를 생성하기 때문). 하지만 `public`으로 열어두면 어디서든 빈 객체를 만들 수 있어 위험하다. `PROTECTED`로 제한하면 JPA는 사용할 수 있지만, 외부에서 `new Member()`는 불가능해진다.

**2. 정적 팩토리 메서드 (`Member.create(...)`):**
생성자 대신 의미 있는 이름의 정적 메서드를 사용한다. 이는:
- 객체 생성 시 반드시 필요한 값들을 강제할 수 있다
- 기본값(role=MEMBER, status=ACTIVE)을 메서드 안에서 처리한다
- `new Member(email, password, name, role, status)` 대신 `Member.create(email, password, name)` → 파라미터 순서 실수 방지

**3. Setter를 만들지 않는다:**
JPA 엔티티에 setter를 열어두면 어디서든 `member.setRole(ADMIN)`을 호출할 수 있다. 대신 **의미 있는 비즈니스 메서드**를 만든다:
```java
public void changeRole(MemberRole newRole) {
    if (this.status != MemberStatus.ACTIVE) {
        throw new BusinessException(ErrorCode.MEMBER_NOT_ACTIVE);
    }
    this.role = newRole;
}
```

**4. `@Enumerated(EnumType.STRING)`:**
Enum을 DB에 저장할 때 두 가지 옵션이 있다:
- `EnumType.ORDINAL` (기본값): 숫자로 저장 (0, 1, 2...). **절대 쓰지 마라!** Enum 순서가 변경되면 데이터가 오염됨
- `EnumType.STRING`: 문자열로 저장 ("ADMIN", "MANAGER"). 안전하고 DB에서 직접 조회할 때도 가독성이 좋음

**5. `@Column(name = "member_id")`:**
Java에서는 `id`라고 쓰지만, DB 컬럼은 `member_id`로 명시한다. 이유:
- DB에서 JOIN 할 때 `id`만으로는 어떤 테이블의 ID인지 모호하다
- `member_id`로 FK를 참조할 때 명확하다

### Issue 엔티티 (복잡한 연관관계 포함)

```java
@Entity
@Table(name = "issues")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Issue extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "issue_id")
    private Long id;

    @Column(nullable = false, length = 200)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private IssueType type; // BUG, FEATURE, IMPROVEMENT, TASK

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private IssuePriority priority; // CRITICAL, HIGH, MEDIUM, LOW

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private IssueStatus status; // TODO, IN_PROGRESS, IN_REVIEW, DONE, CANCELED

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "project_id", nullable = false)
    private Project project;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reporter_id", nullable = false)
    private Member reporter;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "assignee_id")
    private Member assignee; // nullable - 담당자 미배정 가능

    @OneToMany(mappedBy = "issue", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();

    @Version // 낙관적 락을 위한 버전 필드 (Phase 8에서 상세)
    private Long version;

    // === 비즈니스 메서드 === //
    public IssueStatus changeStatus(IssueStatus newStatus) {
        IssueStatus oldStatus = this.status;
        this.status = newStatus;
        return oldStatus; // AOP에서 이전 상태를 이력에 기록하기 위해 반환
    }

    public void assignTo(Member assignee) {
        this.assignee = assignee;
    }
}
```

#### 연관관계 매핑 상세 설명

**`@ManyToOne(fetch = FetchType.LAZY)`:**
- Issue(N) → Project(1): 여러 이슈가 하나의 프로젝트에 속한다
- `@ManyToOne`의 기본 fetch 전략은 **EAGER**이다! → 반드시 `LAZY`로 변경해야 한다
- `@JoinColumn(name = "project_id")`: 이 FK 컬럼이 issues 테이블에 생성된다

**`@OneToMany(mappedBy = "issue")`:**
- Issue(1) → Comment(N): 하나의 이슈에 여러 코멘트가 달린다
- `mappedBy = "issue"`: FK는 Comment 쪽에 있다. Issue는 **읽기 전용으로 조회만** 한다
- `cascade = CascadeType.ALL`: Issue를 저장/삭제하면 연관된 Comment도 함께 저장/삭제
- `orphanRemoval = true`: `comments.remove(comment)`하면 Comment가 DB에서도 삭제된다

**연관관계 편의 메서드:**
양방향 연관관계에서는 **양쪽 모두**를 세팅해야 한다:

```java
// Comment 엔티티에
public void setIssue(Issue issue) {
    this.issue = issue;
    issue.getComments().add(this);
}
```

이 편의 메서드가 없으면:
```java
comment.setIssue(issue);     // Comment → Issue (DB FK 설정)
issue.getComments().add(comment); // Issue → Comment (메모리상 동기화)
// 둘 다 해야 하는데 하나를 빠뜨리는 실수가 잦다
```

## 3.6 Flyway 마이그레이션 SQL

### V1__create_member_table.sql

```sql
CREATE TABLE members (
    member_id    BIGSERIAL PRIMARY KEY,
    email        VARCHAR(100) NOT NULL UNIQUE,
    password     VARCHAR(255) NOT NULL,
    name         VARCHAR(50)  NOT NULL,
    role         VARCHAR(20)  NOT NULL DEFAULT 'MEMBER',
    status       VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    created_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by   VARCHAR(100),
    updated_by   VARCHAR(100)
);

CREATE INDEX idx_members_email ON members(email);
CREATE INDEX idx_members_status ON members(status);
```

### V2__create_project_table.sql

```sql
CREATE TABLE projects (
    project_id   BIGSERIAL PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    description  TEXT,
    project_key  VARCHAR(10)  NOT NULL UNIQUE,  -- 예: "TSB", "PRJ"
    status       VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    created_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by   VARCHAR(100),
    updated_by   VARCHAR(100)
);

CREATE TABLE project_members (
    project_member_id BIGSERIAL PRIMARY KEY,
    project_id   BIGINT      NOT NULL REFERENCES projects(project_id),
    member_id    BIGINT      NOT NULL REFERENCES members(member_id),
    project_role VARCHAR(20) NOT NULL DEFAULT 'DEVELOPER',
    joined_at    TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(project_id, member_id)
);

CREATE INDEX idx_project_members_project ON project_members(project_id);
CREATE INDEX idx_project_members_member ON project_members(member_id);
```

### V3__create_issue_table.sql

```sql
CREATE TABLE issues (
    issue_id     BIGSERIAL PRIMARY KEY,
    title        VARCHAR(200) NOT NULL,
    description  TEXT,
    type         VARCHAR(20)  NOT NULL,
    priority     VARCHAR(20)  NOT NULL DEFAULT 'MEDIUM',
    status       VARCHAR(20)  NOT NULL DEFAULT 'TODO',
    project_id   BIGINT       NOT NULL REFERENCES projects(project_id),
    reporter_id  BIGINT       NOT NULL REFERENCES members(member_id),
    assignee_id  BIGINT       REFERENCES members(member_id),
    version      BIGINT       NOT NULL DEFAULT 0,
    created_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by   VARCHAR(100),
    updated_by   VARCHAR(100)
);

CREATE INDEX idx_issues_project ON issues(project_id);
CREATE INDEX idx_issues_status ON issues(status);
CREATE INDEX idx_issues_assignee ON issues(assignee_id);
CREATE INDEX idx_issues_reporter ON issues(reporter_id);

CREATE TABLE issue_histories (
    issue_history_id BIGSERIAL PRIMARY KEY,
    issue_id     BIGINT       NOT NULL REFERENCES issues(issue_id),
    changed_by   BIGINT       NOT NULL REFERENCES members(member_id),
    field_name   VARCHAR(50)  NOT NULL,
    old_value    VARCHAR(255),
    new_value    VARCHAR(255),
    changed_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_issue_histories_issue ON issue_histories(issue_id);
```

### V4__create_comment_and_label_tables.sql

```sql
CREATE TABLE comments (
    comment_id   BIGSERIAL PRIMARY KEY,
    content      TEXT         NOT NULL,
    issue_id     BIGINT       NOT NULL REFERENCES issues(issue_id) ON DELETE CASCADE,
    author_id    BIGINT       NOT NULL REFERENCES members(member_id),
    created_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by   VARCHAR(100),
    updated_by   VARCHAR(100)
);

CREATE INDEX idx_comments_issue ON comments(issue_id);

CREATE TABLE labels (
    label_id     BIGSERIAL PRIMARY KEY,
    name         VARCHAR(50)  NOT NULL,
    color        VARCHAR(7)   NOT NULL DEFAULT '#6B7280',
    project_id   BIGINT       NOT NULL REFERENCES projects(project_id),
    UNIQUE(name, project_id)
);

CREATE TABLE issue_labels (
    issue_id     BIGINT NOT NULL REFERENCES issues(issue_id) ON DELETE CASCADE,
    label_id     BIGINT NOT NULL REFERENCES labels(label_id) ON DELETE CASCADE,
    PRIMARY KEY (issue_id, label_id)
);
```

## 3.7 Phase 3 Claude Code 구현 프롬프트

### 구현 프롬프트

```
TaskSyncBoard Phase 3: 도메인 모델링 + JPA + Flyway를 구현해줘.

★ 먼저 build.gradle에 아래 의존성을 추가해줘:
   - spring-boot-starter-data-jpa
   - postgresql (runtimeOnly)
   - flyway-core
   - flyway-database-postgresql

★ application-local.yml에 DB 연결, JPA, Flyway 설정을 추가해줘.

그 다음:

1. JPA Auditing 설정:
   - BaseEntity 추상 클래스 생성 (createdAt, updatedAt, createdBy, updatedBy)
   - JpaAuditingConfig 생성 (@EnableJpaAuditing + AuditorAware<String> Bean)
   - AuditorAware는 임시로 "SYSTEM" 반환 (Phase 5에서 Security 추가 후 교체)

2. Enum 클래스 생성:
   - MemberRole (ADMIN, MANAGER, MEMBER)
   - MemberStatus (ACTIVE, INACTIVE, WITHDRAWN)
   - ProjectStatus (ACTIVE, ARCHIVED, DELETED)
   - ProjectRole (LEADER, MAINTAINER, DEVELOPER, VIEWER)
   - IssueType (BUG, FEATURE, IMPROVEMENT, TASK)
   - IssuePriority (CRITICAL, HIGH, MEDIUM, LOW)
   - IssueStatus (TODO, IN_PROGRESS, IN_REVIEW, DONE, CANCELED)

3. Entity 클래스 생성:
   - Member, Project, ProjectMember, Issue, IssueHistory, Comment, Label
   모든 엔티티는:
   - BaseEntity 상속 (IssueHistory, Label 제외)
   - @NoArgsConstructor(access = PROTECTED)
   - Setter 없이 비즈니스 메서드 + 정적 팩토리 메서드
   - 모든 @ManyToOne은 fetch = FetchType.LAZY
   - Issue 엔티티에 @Version 필드 포함

4. Repository 인터페이스 생성:
   - MemberRepository (findByEmail 추가)
   - ProjectRepository, ProjectMemberRepository
   - IssueRepository, IssueHistoryRepository, CommentRepository, LabelRepository

5. Flyway 마이그레이션 SQL 파일 생성:
   - V1 ~ V4 (위 가이드 참조)

6. Issue와 Label 사이의 @ManyToMany 매핑 추가
```

### 확인 및 테스트 프롬프트

```
Phase 3 구현이 올바른지 확인해줘:

1. Docker Compose로 PostgreSQL이 실행 중인지 확인해줘.
2. './gradlew bootRun' 으로 애플리케이션을 시작해줘.
   - Flyway 마이그레이션이 성공적으로 실행되는지 확인
   - JPA validate가 에러를 내지 않는지 확인
3. PostgreSQL에 접속해서 테이블이 생성되었는지 확인:
   docker exec tsb-postgres psql -U tsb_user -d tasksyncboard -c "\dt"
4. flyway_schema_history 확인:
   docker exec tsb-postgres psql -U tsb_user -d tasksyncboard -c "SELECT * FROM flyway_schema_history"
5. 모든 @ManyToOne에 fetch = FetchType.LAZY가 설정되어 있는지 확인.
6. 에러가 있으면 원인을 분석하고 수정해줘.
```

---

# Phase 4: API 개발 + Validation + 전역 예외 처리

## 이 Phase에서 추가하는 의존성

| 라이브러리 | 용도 |
|-----------|------|
| `spring-boot-starter-validation` | Bean Validation (Jakarta Validation) - @NotBlank, @Size, @Email 등 |

### build.gradle에 추가할 내용

```groovy
dependencies {
    // === Phase 1 ~ 3 (기존) === //
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql'
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    // === Phase 4: Validation === //
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

> **왜 이제 Validation을 추가하는가?** API를 만들면서 Request DTO에 입력값 검증이 필요하기 때문이다. @NotBlank, @Size, @Email 같은 어노테이션은 이 라이브러리가 있어야 쓸 수 있다.

## 4.1 Spring MVC 요청 처리 흐름 딥다이브

> **이 섹션이 Phase 4에 있는 이유:** REST API를 본격적으로 개발하기 시작하므로, HTTP 요청이 Spring Boot 내부에서 어떻게 처리되는지 여기서 이해한다.

HTTP 요청이 들어왔을 때 Spring Boot 내부에서 어떤 일이 벌어지는지 **완전히** 이해해야 한다.

```
클라이언트 HTTP 요청
       │
       ▼
┌──────────────┐
│  Tomcat       │  ← Embedded Servlet Container
│  (NIO Connector)  서블릿 스펙에 따라 요청을 받는다
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Filter Chain │  ← Spring Security 필터가 여기서 동작! (Phase 5에서 추가)
│  (서블릿 필터)  │  CharacterEncodingFilter → ...
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│  DispatcherServlet    │  ← Spring MVC의 "프론트 컨트롤러"
│  (서블릿)              │  모든 요청의 진입점. 적절한 핸들러를 찾아 위임한다
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  HandlerMapping       │  요청 URL + HTTP 메서드 → 어떤 Controller의 어떤 메서드?
│                       │  @RequestMapping 정보를 기반으로 탐색
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  HandlerAdapter       │  Controller 메서드를 실제로 호출하는 역할
│                       │  @RequestBody → JSON 역직렬화 (Jackson)
│                       │  @Valid → Validation 검증
│                       │  @PathVariable, @RequestParam → 파라미터 바인딩
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  Controller 메서드     │  비즈니스 로직 호출
│  → Service            │
│  → Repository         │
│  → DB                 │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  HandlerAdapter       │  반환값 처리
│                       │  @ResponseBody → 객체를 JSON으로 직렬화 (Jackson)
│                       │  ResponseEntity → 상태 코드 + 헤더 + 바디 설정
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  예외 발생 시:          │
│  @RestControllerAdvice │  ← 전역 예외 핸들러가 여기서 동작
│  (ExceptionHandler)    │
└──────┬───────────────┘
       │
       ▼
클라이언트 HTTP 응답
```

## 4.2 일관된 API 응답 구조 설계

모든 API의 응답 형식이 다르면 프론트엔드 개발자가 혼란을 겪는다. **성공이든 실패든 동일한 구조**로 응답해야 한다.

### 성공 응답

```json
{
    "status": "SUCCESS",
    "data": {
        "issueId": 1,
        "title": "로그인 버그 수정",
        "status": "TODO"
    },
    "message": null
}
```

### 에러 응답

```json
{
    "status": "ERROR",
    "data": null,
    "message": "해당 이슈를 찾을 수 없습니다.",
    "errorCode": "ISSUE_NOT_FOUND",
    "errors": []
}
```

### Validation 에러 응답

```json
{
    "status": "ERROR",
    "data": null,
    "message": "입력값이 올바르지 않습니다.",
    "errorCode": "INVALID_INPUT",
    "errors": [
        {
            "field": "title",
            "value": "",
            "reason": "이슈 제목은 필수입니다."
        },
        {
            "field": "type",
            "value": null,
            "reason": "이슈 타입은 필수입니다."
        }
    ]
}
```

### 구현

```java
// === 공통 응답 래퍼 === //
@Getter
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class ApiResponse<T> {
    private final String status;
    private final T data;
    private final String message;

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>("SUCCESS", data, null);
    }

    public static ApiResponse<Void> success() {
        return new ApiResponse<>("SUCCESS", null, null);
    }

    public static ApiResponse<Void> error(String message) {
        return new ApiResponse<>("ERROR", null, message);
    }
}
```

## 4.3 전역 예외 처리 (Global Exception Handling) 딥다이브

### 왜 전역 예외 처리가 필요한가?

예외 처리가 없으면 Spring은 기본적으로 HTML 에러 페이지를 반환한다. REST API에서는 이것이 의미 없다.

각 Controller마다 `try-catch`를 넣으면 모든 컨트롤러에 같은 코드가 반복된다. **@RestControllerAdvice**가 이 중복을 제거한다.

### 설계

```java
// === 에러 코드 Enum === //
@Getter
@AllArgsConstructor
public enum ErrorCode {
    // Common
    INVALID_INPUT(HttpStatus.BAD_REQUEST, "C001", "입력값이 올바르지 않습니다."),
    INTERNAL_SERVER_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "C002", "서버 내부 에러가 발생했습니다."),
    RESOURCE_NOT_FOUND(HttpStatus.NOT_FOUND, "C003", "요청한 리소스를 찾을 수 없습니다."),
    METHOD_NOT_ALLOWED(HttpStatus.METHOD_NOT_ALLOWED, "C004", "허용되지 않은 HTTP 메서드입니다."),
    ACCESS_DENIED(HttpStatus.FORBIDDEN, "C005", "접근 권한이 없습니다."),

    // Member
    MEMBER_NOT_FOUND(HttpStatus.NOT_FOUND, "M001", "해당 회원을 찾을 수 없습니다."),
    DUPLICATE_EMAIL(HttpStatus.CONFLICT, "M002", "이미 사용 중인 이메일입니다."),
    MEMBER_NOT_ACTIVE(HttpStatus.BAD_REQUEST, "M003", "비활성 상태의 회원입니다."),

    // Auth (Phase 5에서 사용)
    INVALID_TOKEN(HttpStatus.UNAUTHORIZED, "A001", "유효하지 않은 토큰입니다."),
    EXPIRED_TOKEN(HttpStatus.UNAUTHORIZED, "A002", "만료된 토큰입니다."),
    INVALID_CREDENTIALS(HttpStatus.UNAUTHORIZED, "A003", "이메일 또는 비밀번호가 올바르지 않습니다."),
    REFRESH_TOKEN_NOT_FOUND(HttpStatus.UNAUTHORIZED, "A004", "Refresh Token이 존재하지 않습니다."),

    // Project
    PROJECT_NOT_FOUND(HttpStatus.NOT_FOUND, "P001", "해당 프로젝트를 찾을 수 없습니다."),
    DUPLICATE_PROJECT_KEY(HttpStatus.CONFLICT, "P002", "이미 사용 중인 프로젝트 키입니다."),
    NOT_PROJECT_MEMBER(HttpStatus.FORBIDDEN, "P003", "해당 프로젝트의 멤버가 아닙니다."),
    INSUFFICIENT_PROJECT_ROLE(HttpStatus.FORBIDDEN, "P004", "해당 작업에 필요한 프로젝트 권한이 없습니다."),

    // Issue
    ISSUE_NOT_FOUND(HttpStatus.NOT_FOUND, "I001", "해당 이슈를 찾을 수 없습니다."),
    INVALID_STATUS_TRANSITION(HttpStatus.BAD_REQUEST, "I002", "유효하지 않은 상태 전환입니다."),
    ISSUE_ALREADY_MODIFIED(HttpStatus.CONFLICT, "I003", "이슈가 다른 사용자에 의해 이미 수정되었습니다."),

    // Comment
    COMMENT_NOT_FOUND(HttpStatus.NOT_FOUND, "CM001", "해당 코멘트를 찾을 수 없습니다."),
    NOT_COMMENT_AUTHOR(HttpStatus.FORBIDDEN, "CM002", "코멘트 작성자만 수정/삭제할 수 있습니다.");

    private final HttpStatus httpStatus;
    private final String code;
    private final String message;
}

// === 커스텀 비즈니스 예외 === //
@Getter
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }

    public BusinessException(ErrorCode errorCode, String customMessage) {
        super(customMessage);
        this.errorCode = errorCode;
    }
}

// === 에러 응답 DTO === //
@Getter
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class ErrorResponse {
    private final String status;
    private final String errorCode;
    private final String message;
    private final List<FieldError> errors;

    @Getter
    @AllArgsConstructor
    public static class FieldError {
        private final String field;
        private final String value;
        private final String reason;
    }

    public static ErrorResponse of(ErrorCode errorCode) {
        return new ErrorResponse("ERROR", errorCode.getCode(), errorCode.getMessage(), List.of());
    }

    public static ErrorResponse of(ErrorCode errorCode, List<FieldError> errors) {
        return new ErrorResponse("ERROR", errorCode.getCode(), errorCode.getMessage(), errors);
    }

    public static ErrorResponse of(ErrorCode errorCode, BindingResult bindingResult) {
        List<FieldError> fieldErrors = bindingResult.getFieldErrors().stream()
                .map(error -> new FieldError(
                        error.getField(),
                        error.getRejectedValue() == null ? "" : error.getRejectedValue().toString(),
                        error.getDefaultMessage()))
                .toList();
        return new ErrorResponse("ERROR", errorCode.getCode(), errorCode.getMessage(), fieldErrors);
    }
}

// === 전역 예외 핸들러 === //
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    protected ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        log.warn("BusinessException: {}", e.getMessage());
        ErrorCode errorCode = e.getErrorCode();
        return ResponseEntity.status(errorCode.getHttpStatus())
                .body(ErrorResponse.of(errorCode));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    protected ResponseEntity<ErrorResponse> handleValidationException(
            MethodArgumentNotValidException e) {
        log.warn("Validation failed: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(ErrorResponse.of(ErrorCode.INVALID_INPUT, e.getBindingResult()));
    }

    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    protected ResponseEntity<ErrorResponse> handleTypeMismatchException(
            MethodArgumentTypeMismatchException e) {
        log.warn("Type mismatch: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(ErrorResponse.of(ErrorCode.INVALID_INPUT));
    }

    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    protected ResponseEntity<ErrorResponse> handleMethodNotAllowed(
            HttpRequestMethodNotSupportedException e) {
        log.warn("Method not allowed: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.METHOD_NOT_ALLOWED)
                .body(ErrorResponse.of(ErrorCode.METHOD_NOT_ALLOWED));
    }

    // 낙관적 락 충돌 (Phase 8에서 사용)
    @ExceptionHandler(ObjectOptimisticLockingFailureException.class)
    protected ResponseEntity<ErrorResponse> handleOptimisticLockException(
            ObjectOptimisticLockingFailureException e) {
        log.warn("Optimistic lock conflict: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(ErrorResponse.of(ErrorCode.ISSUE_ALREADY_MODIFIED));
    }

    // 그 외 모든 예외 (안전망)
    @ExceptionHandler(Exception.class)
    protected ResponseEntity<ErrorResponse> handleException(Exception e) {
        log.error("Unhandled exception: ", e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ErrorResponse.of(ErrorCode.INTERNAL_SERVER_ERROR));
    }
}
```

### @RestControllerAdvice 동작 원리

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`

- Spring MVC의 `ExceptionHandlerExceptionResolver`가 예외 발생 시 `@RestControllerAdvice` 클래스들을 탐색
- 예외 타입에 매칭되는 `@ExceptionHandler` 메서드를 찾아 실행
- 반환값이 `@ResponseBody`에 의해 JSON으로 직렬화되어 응답

**핸들러 우선순위:** 가장 구체적인 예외 타입이 우선이다. `BusinessException`과 `Exception` 핸들러가 동시에 매칭되면 `BusinessException` 핸들러가 호출된다.

## 4.4 Bean Validation 딥다이브

### 왜 Validation을 써야 하는가?

서비스 레이어에서 직접 검증하면 검증 코드가 비즈니스 로직보다 길어진다. Bean Validation은 **선언적으로** 검증 규칙을 정의한다:

```java
@Getter
@NoArgsConstructor
public class IssueCreateRequest {

    @NotBlank(message = "이슈 제목은 필수입니다.")
    @Size(max = 200, message = "이슈 제목은 200자 이내여야 합니다.")
    private String title;

    private String description;

    @NotNull(message = "이슈 타입은 필수입니다.")
    private IssueType type;

    @NotNull(message = "이슈 우선순위는 필수입니다.")
    private IssuePriority priority;

    @NotNull(message = "프로젝트 ID는 필수입니다.")
    private Long projectId;

    private Long assigneeId; // nullable (담당자 미배정 가능)
}
```

Controller에서 `@Valid`를 붙이면 자동 검증:
```java
@PostMapping
public ResponseEntity<ApiResponse<IssueResponse>> createIssue(
        @Valid @RequestBody IssueCreateRequest request) {
    // 여기 도달했다면 모든 Validation을 통과한 것!
    IssueResponse response = issueService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success(response));
}
```

### 주요 Validation 어노테이션

| 어노테이션 | 대상 | 동작 |
|-----------|------|------|
| `@NotNull` | 모든 타입 | null 불가 |
| `@NotEmpty` | String, Collection | null 불가 + 비어있으면 안 됨 |
| `@NotBlank` | String | null 불가 + 공백만 있으면 안 됨 |
| `@Size(min, max)` | String, Collection | 크기/길이 제한 |
| `@Min(value)` / `@Max(value)` | 숫자 | 최소/최대값 |
| `@Email` | String | 이메일 형식 검증 |
| `@Pattern(regexp)` | String | 정규식 검증 |
| `@Positive` / `@PositiveOrZero` | 숫자 | 양수 / 0 이상 |

### DTO 패턴: Request DTO와 Response DTO 분리

```
클라이언트 → [Request DTO] → Controller → Service → Repository → DB
                                                         ↓
클라이언트 ← [Response DTO] ← Controller ← Service ← Entity
```

**왜 Entity를 직접 반환하면 안 되는가?**

1. **순환 참조:** Issue → Project → Issues → ... Jackson이 무한 직렬화
2. **보안:** password 같은 민감 정보가 노출될 수 있음
3. **커플링:** API 스펙이 Entity 구조에 종속됨. Entity 변경 시 API가 깨짐
4. **지연 로딩 문제:** Controller에서 Lazy 프록시 접근 시 LazyInitializationException

## 4.5 Controller - Service - Repository 계층 구조

```java
// === Controller (표현 계층) === //
@RestController
@RequestMapping("/api/v1/issues")
@RequiredArgsConstructor
public class IssueController {

    private final IssueService issueService;

    @PostMapping
    public ResponseEntity<ApiResponse<IssueResponse>> createIssue(
            @Valid @RequestBody IssueCreateRequest request) {
        IssueResponse response = issueService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(response));
    }

    @GetMapping("/{issueId}")
    public ResponseEntity<ApiResponse<IssueDetailResponse>> getIssue(
            @PathVariable Long issueId) {
        IssueDetailResponse response = issueService.findById(issueId);
        return ResponseEntity.ok(ApiResponse.success(response));
    }

    @PatchMapping("/{issueId}/status")
    public ResponseEntity<ApiResponse<IssueResponse>> changeStatus(
            @PathVariable Long issueId,
            @Valid @RequestBody IssueStatusChangeRequest request) {
        IssueResponse response = issueService.changeStatus(issueId, request);
        return ResponseEntity.ok(ApiResponse.success(response));
    }

    @GetMapping
    public ResponseEntity<ApiResponse<Page<IssueResponse>>> getIssues(
            @RequestParam Long projectId,
            @RequestParam(required = false) IssueStatus status,
            Pageable pageable) {
        Page<IssueResponse> response = issueService.findByProject(projectId, status, pageable);
        return ResponseEntity.ok(ApiResponse.success(response));
    }
}

// === Service (비즈니스 계층) === //
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true) // 클래스 레벨: 기본적으로 읽기 전용
public class IssueService {

    private final IssueRepository issueRepository;
    private final ProjectRepository projectRepository;
    private final MemberRepository memberRepository;

    @Transactional // 쓰기 작업은 메서드 레벨에서 오버라이드
    public IssueResponse create(IssueCreateRequest request) {
        Project project = projectRepository.findById(request.getProjectId())
                .orElseThrow(() -> new BusinessException(ErrorCode.PROJECT_NOT_FOUND));

        // Phase 5에서 Security를 추가하면 SecurityContext에서 현재 사용자를 가져온다.
        // 지금은 임시로 Member를 직접 조회한다.
        Member reporter = memberRepository.findById(1L)
                .orElseThrow(() -> new BusinessException(ErrorCode.MEMBER_NOT_FOUND));

        Member assignee = null;
        if (request.getAssigneeId() != null) {
            assignee = memberRepository.findById(request.getAssigneeId())
                    .orElseThrow(() -> new BusinessException(ErrorCode.MEMBER_NOT_FOUND));
        }

        Issue issue = Issue.create(request.getTitle(), request.getDescription(),
                request.getType(), request.getPriority(), project, reporter, assignee);

        Issue savedIssue = issueRepository.save(issue);
        return IssueResponse.from(savedIssue);
    }

    public IssueDetailResponse findById(Long issueId) {
        Issue issue = issueRepository.findById(issueId)
                .orElseThrow(() -> new BusinessException(ErrorCode.ISSUE_NOT_FOUND));
        return IssueDetailResponse.from(issue);
    }
}
```

### `@Transactional(readOnly = true)`의 의미

- Hibernate가 **Dirty Checking을 하지 않는다** → 성능 향상 (스냅샷을 만들지 않으므로 메모리 절약)
- DB 드라이버에 "읽기 전용" 힌트를 전달 → DB가 Read Replica로 라우팅할 수 있음
- 클래스 레벨에 `readOnly = true`를 걸고, 쓰기 메서드에만 `@Transactional`을 다시 걸어 오버라이드하는 것이 베스트 프랙티스

> **지금은 Spring Security가 없으므로 모든 API가 인증 없이 접근 가능하다.** Phase 5에서 Security + JWT를 추가한 후 인증이 필요한 API를 보호한다.

## 4.6 Phase 4 Claude Code 구현 프롬프트

### 구현 프롬프트

```
TaskSyncBoard Phase 4: REST API + Validation + 전역 예외 처리를 구현해줘.

★ 먼저 build.gradle에 spring-boot-starter-validation 의존성을 추가해줘.

그 다음:

1. 전역 에러 처리 인프라:
   - ErrorCode enum (위 가이드의 모든 에러 코드 포함)
   - BusinessException 클래스
   - ErrorResponse DTO (FieldError 내부 클래스 포함)
   - GlobalExceptionHandler (@RestControllerAdvice)

2. 공통 응답 DTO:
   - ApiResponse<T> (success/error 정적 팩토리 메서드)

3. Member 도메인 API:
   - MemberSignupRequest (email: @NotBlank @Email, password: @NotBlank @Size(min=8), name: @NotBlank)
   - MemberResponse (id, email, name, role, status, createdAt)
   - MemberService (signup, findById, findAll)
   - MemberController (POST /api/v1/members/signup, GET /api/v1/members/{id}, GET /api/v1/members)
   - 비밀번호는 일단 평문 저장 (Phase 5에서 BCrypt로 교체)

4. Project 도메인 API:
   - ProjectCreateRequest, ProjectResponse, ProjectDetailResponse
   - ProjectService (create, findById, findAll, addMember)
   - ProjectController (POST /api/v1/projects, GET 등)

5. Issue 도메인 API:
   - IssueCreateRequest, IssueUpdateRequest, IssueStatusChangeRequest
   - IssueResponse, IssueDetailResponse
   - IssueService (create, findById, update, changeStatus, findByProject)
   - IssueController (CRUD + 상태 변경 + 목록 조회)

6. Comment 도메인 API:
   - CommentCreateRequest, CommentResponse
   - CommentService, CommentController

모든 Service는 클래스 레벨 @Transactional(readOnly = true), 쓰기 메서드에만 @Transactional.
모든 Response DTO는 from(Entity) 정적 팩토리 메서드로 변환.
모든 Controller는 ApiResponse로 래핑하여 응답.
```

### 확인 및 테스트 프롬프트

```
Phase 4 구현이 올바른지 확인해줘:

1. 빌드 성공 여부: './gradlew build -x test'
2. 애플리케이션 시작 후 아래 API를 curl로 테스트:
   - 회원가입: POST /api/v1/members/signup (정상 + validation 실패 케이스)
   - 프로젝트 생성: POST /api/v1/projects
   - 이슈 생성: POST /api/v1/issues
   - 이슈 목록 조회: GET /api/v1/issues?projectId=1&page=0&size=10
   - 이슈 상태 변경: PATCH /api/v1/issues/1/status
   - 코멘트 생성: POST /api/v1/issues/1/comments
3. Validation 에러 시 ErrorResponse 형식이 올바른지 확인.
4. 존재하지 않는 리소스 접근 시 적절한 에러 응답이 오는지 확인.
```

---

# Phase 5: Spring Security + JWT + Redis 인증

## 이 Phase에서 추가하는 의존성

| 라이브러리 | 용도 |
|-----------|------|
| `spring-boot-starter-security` | Spring Security 프레임워크 |
| `spring-boot-starter-data-redis` | Redis 클라이언트 (Refresh Token 저장) |
| `io.jsonwebtoken:jjwt-api:0.12.6` | JWT 생성/파싱 API |
| `io.jsonwebtoken:jjwt-impl:0.12.6` | JWT 구현체 (runtime) |
| `io.jsonwebtoken:jjwt-jackson:0.12.6` | JWT Jackson 직렬화 (runtime) |
| `spring-security-test` | Security 테스트 유틸 (test scope) |

### build.gradle에 추가할 내용

```groovy
dependencies {
    // === Phase 1 ~ 4 (기존) === //
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    runtimeOnly 'org.postgresql:postgresql'
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    // === Phase 5: Security + JWT + Redis === //
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'io.jsonwebtoken:jjwt-api:0.12.6'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.6'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.6'
    testImplementation 'org.springframework.security:spring-security-test'
}
```

> **왜 이제 Security + JWT + Redis를 추가하는가?** Phase 4에서 기본 CRUD API가 동작하니, 이제 "누가 이 API를 호출하는가"를 제어해야 한다. Spring Security로 인증/인가를 처리하고, JWT로 토큰 기반 인증을 구현하며, Redis에 Refresh Token을 저장한다.

> **주의:** `spring-boot-starter-security`를 추가하는 순간, **모든 API가 기본적으로 차단된다** (Spring Security의 기본 동작). 따라서 SecurityConfig에서 적절히 설정해야 한다.

### application-local.yml에 추가할 설정

```yaml
# === Phase 5에서 추가 === #
spring:
  data:
    redis:
      host: localhost
      port: 6379

jwt:
  secret: dGhpcyBpcyBhIHZlcnkgbG9uZyBzZWNyZXQga2V5IGZvciBIUzI1NiBhbGdvcml0aG0gYXQgbGVhc3QgMjU2IGJpdHM=
  access-token-expiry: 1800000    # 30분 (밀리초)
  refresh-token-expiry: 1209600000 # 14일 (밀리초)
```

## 5.1 Spring Security 아키텍처 딥다이브

> **이 섹션이 Phase 5에 있는 이유:** Spring Security를 지금 처음 사용하므로, 필터 체인과 인증 아키텍처를 여기서 이해한다.

Spring Security는 **서블릿 필터 체인(Servlet Filter Chain)** 기반으로 동작한다. 이것을 이해하지 않으면 Security 설정이 마법처럼 느껴진다.

### 서블릿 필터와 Spring Security 필터

```
HTTP 요청
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Servlet Container (Tomcat) Filter Chain                     │
│                                                              │
│  ┌──────────────────┐                                       │
│  │ CharacterEncoding │                                       │
│  │ Filter            │                                       │
│  └────────┬─────────┘                                       │
│           ▼                                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ DelegatingFilterProxy                                 │   │
│  │  └─ FilterChainProxy (springSecurityFilterChain)      │   │
│  │       │                                               │   │
│  │       ▼                                               │   │
│  │  ┌──────────────────────────────────────────────┐    │   │
│  │  │ SecurityFilterChain                           │    │   │
│  │  │                                               │    │   │
│  │  │ 1. SecurityContextPersistenceFilter           │    │   │
│  │  │ 2. CorsFilter                                 │    │   │
│  │  │ 3. CsrfFilter                                 │    │   │
│  │  │ 4. LogoutFilter                               │    │   │
│  │  │ 5. UsernamePasswordAuthenticationFilter        │    │   │
│  │  │ 6. ★ JwtAuthenticationFilter (우리가 추가!)    │    │   │
│  │  │ 7. ExceptionTranslationFilter                 │    │   │
│  │  │ 8. FilterSecurityInterceptor                  │    │   │
│  │  └──────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────┘   │
│           ▼                                                  │
│  ┌──────────────────┐                                       │
│  │ DispatcherServlet │                                       │
│  └──────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
```

### 핵심 개념

**SecurityContext와 Authentication:**
```java
// SecurityContext는 "현재 인증된 사용자 정보"를 담는 컨테이너
SecurityContext context = SecurityContextHolder.getContext();

// Authentication 객체가 "누가, 어떤 권한으로" 인증되었는지를 담는다
Authentication auth = context.getAuthentication();
auth.getPrincipal();     // 인증된 사용자 (UserDetails 구현체)
auth.getAuthorities();   // 사용자의 권한 목록 (ROLE_ADMIN 등)
auth.isAuthenticated();  // 인증 여부
```

**SecurityContextHolder의 전략:**
- `MODE_THREADLOCAL` (기본): 각 스레드의 ThreadLocal에 SecurityContext를 저장. 같은 스레드 내에서는 어디서든 `SecurityContextHolder.getContext()`로 인증 정보에 접근 가능.
- 요청이 끝나면 SecurityContext는 자동으로 정리된다.

## 5.2 JWT (JSON Web Token) 딥다이브

### 왜 JWT인가? (세션 vs JWT)

**세션 기반 인증:**
- 로그인 → 서버가 세션 ID 생성 → 서버 메모리(or Redis)에 세션 저장
- 문제: 서버가 **상태(State)를 유지**해야 한다. 서버가 여러 대이면 세션 동기화 문제 발생.

**JWT 기반 인증:**
- 로그인 → 서버가 JWT 발급 → 클라이언트가 저장 → 매 요청마다 Authorization 헤더로 전송
- 장점: 서버가 **무상태(Stateless)**. 서명 검증만 하면 되므로 어떤 서버가 받아도 처리 가능.

### JWT 구조

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.    ← Header (Base64)
eyJzdWIiOiJ1c2VyQGVtYWlsLmNvbSIsInJvbGUiOiJNRU1CRVIiLCJpYXQiOjE3MDk4ODAwMDB9.  ← Payload (Base64)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c    ← Signature
```

**Header:** `{ "alg": "HS256", "typ": "JWT" }`

**Payload (Claims):**
```json
{
    "sub": "user@email.com",      // subject: 사용자 식별자
    "role": "MEMBER",             // 커스텀 클레임: 역할
    "memberId": 1,                // 커스텀 클레임: 사용자 ID
    "iat": 1709880000,            // issued at: 발급 시각
    "exp": 1709883600             // expiration: 만료 시각
}
```

**Signature:** Header + Payload + Secret Key로 생성한 서명

> **주의:** Payload는 **암호화되지 않는다!** Base64로 디코딩하면 누구나 읽을 수 있다. 따라서 비밀번호 같은 민감 정보를 Payload에 담으면 안 된다.

### Access Token + Refresh Token 전략

**왜 토큰을 두 종류로 나누는가?**

JWT는 한번 발급하면 서버에서 임의로 무효화할 수 없다(무상태이므로). Access Token 유효 기간이 길면 탈취 시 악용 위험이 크다.

해결: **짧은 수명의 Access Token + 긴 수명의 Refresh Token**

- **Access Token:** 수명 30분, API 요청 인증용, 서버에 저장하지 않음
- **Refresh Token:** 수명 14일, Access Token 재발급용, **Redis에 저장** (로그아웃 시 삭제 가능)

**인증 흐름:**
```
1. [로그인] → Access Token + Refresh Token 발급, RT를 Redis에 저장
2. [API 요청] → Access Token으로 인증
3. [AT 만료] → 401 응답
4. [토큰 재발급] → Refresh Token으로 새 토큰 쌍 발급 (Rotation)
5. [로그아웃] → Redis에서 RT 삭제
```

### Redis에 Refresh Token 저장하는 이유

1. **빠른 조회:** Redis는 인메모리 DB로 O(1) 조회
2. **TTL 자동 만료:** `SET key value EX 1209600` (14일 후 자동 삭제)
3. **로그아웃 처리:** Redis에서 삭제하면 Refresh Token이 즉시 무효화
4. **서버 재시작 독립:** 외부 저장소이므로 서버 재시작에 영향 없음

## 5.3 SecurityConfig / JwtTokenProvider / JwtAuthenticationFilter 설계

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtTokenProvider jwtTokenProvider;
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    private final JwtAccessDeniedHandler jwtAccessDeniedHandler;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .exceptionHandling(exception -> exception
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                .accessDeniedHandler(jwtAccessDeniedHandler))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(
                    "/api/v1/auth/**",
                    "/api/v1/members/signup",
                    "/swagger-ui/**",
                    "/v3/api-docs/**",
                    "/actuator/health"
                ).permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(new JwtAuthenticationFilter(jwtTokenProvider),
                    UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("http://localhost:3000"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

**각 설정의 이유:**

- **`csrf.disable()`:** JWT는 Authorization 헤더로 전송하므로 CSRF 공격에 해당하지 않는다.
- **`SessionCreationPolicy.STATELESS`:** 세션을 생성하지도, 사용하지도 않겠다는 선언.
- **`addFilterBefore(...)`:** JWT 필터를 기본 인증 필터 앞에 배치한다.

```java
@Slf4j
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String token = resolveToken(request);

        if (token != null && jwtTokenProvider.validateToken(token)) {
            Authentication authentication = jwtTokenProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }

    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

## 5.4 Phase 5 Claude Code 구현 프롬프트

### 구현 프롬프트

```
TaskSyncBoard Phase 5: Spring Security + JWT + Redis 인증을 구현해줘.

★ 먼저 build.gradle에 아래 의존성을 추가해줘:
   - spring-boot-starter-security
   - spring-boot-starter-data-redis
   - jjwt-api:0.12.6, jjwt-impl:0.12.6 (runtimeOnly), jjwt-jackson:0.12.6 (runtimeOnly)
   - spring-security-test (testImplementation)

★ application-local.yml에 Redis 연결 설정과 JWT 설정을 추가해줘.

그 다음:

1. Redis 설정:
   - RedisConfig: RedisTemplate<String, String> Bean 등록
   - RefreshTokenService: Redis에 Refresh Token 저장/조회/삭제

2. JWT 관련 클래스:
   - JwtTokenProvider, JwtAuthenticationFilter
   - JwtAuthenticationEntryPoint (401), JwtAccessDeniedHandler (403)
   - CustomUserPrincipal

3. SecurityConfig:
   - CSRF 비활성화, 세션 Stateless, CORS 설정, JWT 필터 등록
   - URL별 접근 권한 설정

4. Auth API:
   - AuthController: login, reissue, logout
   - AuthService: 로그인, 토큰 재발급 (Rotation), 로그아웃

5. 기존 코드 수정:
   - MemberService: BCryptPasswordEncoder로 비밀번호 암호화
   - JpaAuditingConfig: AuditorAware를 SecurityContext에서 실제 사용자 가져오도록 수정
   - SecurityUtils 유틸 클래스: getCurrentMemberId(), getCurrentEmail()
```

### 확인 및 테스트 프롬프트

```
Phase 5 인증 시스템이 올바르게 동작하는지 확인해줘:

1. 빌드 성공: './gradlew build -x test'
2. Docker Compose (PostgreSQL + Redis) 실행 확인
3. 시나리오 테스트 (curl):
   a. 회원가입: POST /api/v1/members/signup → 성공
   b. 인증 없이 이슈 목록 조회: GET /api/v1/issues → 401 응답 확인
   c. 로그인: POST /api/v1/auth/login → AT + RT 발급
   d. AT으로 API 호출: GET /api/v1/issues → 200 성공
   e. 잘못된 토큰으로 API 호출 → 401
   f. 토큰 재발급: POST /api/v1/auth/reissue → 새 토큰 쌍
   g. 로그아웃: POST /api/v1/auth/logout → Redis RT 삭제
   h. 로그아웃 후 RT로 재발급 시도 → 401
4. Redis 확인: docker exec tsb-redis redis-cli keys "RT:*"
```

---

# Phase 6: AOP 마스터 - 이력 추적 및 권한 체크

## 이 Phase에서 추가하는 의존성

**없음.** Spring AOP는 `spring-boot-starter-web`에 이미 포함되어 있다 (`spring-boot-starter-aop`가 자동으로 들어온다). 별도 의존성 추가 없이 `@Aspect`를 바로 사용할 수 있다.

## 6.1 AOP(Aspect-Oriented Programming) 딥다이브

> **이 섹션이 Phase 6에 있는 이유:** AOP를 지금 처음 사용하므로, AOP의 개념과 프록시 동작 원리를 여기서 이해한다.

### AOP가 해결하는 문제

애플리케이션의 관심사(Concern)를 두 종류로 나눌 수 있다:

1. **핵심 관심사(Core Concern):** 비즈니스 로직
2. **횡단 관심사(Cross-Cutting Concern):** 여러 비즈니스 로직에 공통적으로 필요한 기능 (로깅, 보안, 트랜잭션, 이력 추적 등)

횡단 관심사를 비즈니스 로직에 직접 넣으면 비즈니스 로직이 로깅/권한/이력 코드에 묻혀 가독성이 저하된다.

**AOP를 적용하면:**
```java
@Transactional
@TrackHistory  // 커스텀 어노테이션: 이력 자동 저장
@CheckProjectRole(minRole = ProjectRole.DEVELOPER)  // 권한 자동 체크
public IssueResponse changeStatus(Long issueId, IssueStatusChangeRequest request) {
    // 순수한 비즈니스 로직만!
    Issue issue = issueRepository.findById(issueId)
            .orElseThrow(() -> new BusinessException(ErrorCode.ISSUE_NOT_FOUND));
    IssueStatus oldStatus = issue.changeStatus(request.getStatus());
    return IssueResponse.from(issue);
}
```

### AOP 핵심 용어

| 용어 | 설명 | 예시 |
|------|------|------|
| **Aspect** | 횡단 관심사를 모듈화한 클래스 | `IssueHistoryAspect` |
| **Advice** | "언제" 부가 기능을 실행할 것인가 | `@Before`, `@After`, `@Around` 등 |
| **Join Point** | Advice가 적용될 수 있는 지점 | 메서드 실행 |
| **Pointcut** | "어디에" 적용할 것인가 | `@annotation(TrackHistory)` |
| **Target** | 부가 기능이 적용될 실제 객체 | `IssueService` |
| **Proxy** | Target을 감싸서 부가 기능을 제공하는 객체 | Spring이 자동 생성 |

### Advice 종류 상세

```java
@Aspect
@Component
public class ExampleAspect {

    @Before("@annotation(TrackHistory)")       // 메서드 실행 전
    @AfterReturning(returning = "result")       // 메서드 정상 반환 후
    @AfterThrowing(throwing = "ex")             // 예외 발생 후
    @After("@annotation(TrackHistory)")         // 항상 (finally)
    @Around("@annotation(TrackHistory)")        // 실행 전후를 모두 감싸는 가장 강력한 Advice
}
```

### Spring AOP의 프록시 동작 원리

```
┌───────────────────────────────────────────────────────────────┐
│  IssueController가 주입받는 것은 실제 IssueService가 아니라     │
│  Spring이 생성한 "프록시 객체"이다!                               │
│                                                                │
│  changeStatus() 호출 시:                                       │
│  1. @CheckProjectRole Aspect 실행 (Before) → 권한 체크          │
│  2. @Transactional 처리 (트랜잭션 시작)                          │
│  3. 실제 IssueService.changeStatus() 호출 (비즈니스 로직)       │
│  4. @TrackHistory Aspect 실행 (AfterReturning) → 이력 저장      │
│  5. 트랜잭션 커밋                                               │
└───────────────────────────────────────────────────────────────┘
```

> **핵심 기억:** Spring AOP의 프록시는 **외부 호출에만 동작**한다. 같은 클래스 내에서 `this.method()`로 호출하면 프록시를 거치지 않으므로 AOP가 적용되지 않는다.

## 6.2 커스텀 어노테이션 + AOP 구현: 이슈 변경 이력 자동 저장

```java
// === 커스텀 어노테이션 === //
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TrackHistory {
    String entityType() default "ISSUE";
}

// === 상태 비교용 스냅샷 Record === //
public record IssueSnapshot(String title, String status, String priority, Long assigneeId) {
    public static IssueSnapshot from(Issue issue) {
        return new IssueSnapshot(
            issue.getTitle(), issue.getStatus().name(), issue.getPriority().name(),
            issue.getAssignee() != null ? issue.getAssignee().getId() : null
        );
    }
}

// === Aspect 구현 === //
@Slf4j
@Aspect
@Component
@Order(2) // ProjectRoleCheckAspect(1)보다 나중에 실행
@RequiredArgsConstructor
public class IssueHistoryAspect {
    // @Around로 변경 전/후 스냅샷 비교하여 이력 저장
    // (상세 구현은 가이드 본문 참조)
}
```

## 6.3 커스텀 어노테이션 + AOP 구현: 프로젝트 권한 체크

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CheckProjectRole {
    ProjectRole minRole();
}

// ProjectRole enum에 level 필드 추가
public enum ProjectRole {
    VIEWER(1), DEVELOPER(2), MAINTAINER(3), LEADER(4);
    private final int level;
    // ...
}
```

## 6.4 Phase 6 Claude Code 구현 프롬프트

### 구현 프롬프트

```
TaskSyncBoard Phase 6: AOP를 활용한 이력 추적 + 권한 체크를 구현해줘.

(build.gradle 수정 불필요 - Spring AOP는 이미 포함되어 있음)

1. 커스텀 어노테이션: @TrackHistory, @CheckProjectRole, @ProjectId
2. IssueSnapshot Record
3. IssueHistoryAspect (@Aspect, @Order(2)): @Around로 변경 전/후 비교 → 이력 저장
4. ProjectRoleCheckAspect (@Aspect, @Order(1)): @Before로 권한 체크
5. ProjectRole enum에 level 필드 추가
6. ProjectMember에 hasMinimumRole() 메서드 추가
7. SecurityUtils 유틸 클래스
8. IssueService 메서드에 @TrackHistory + @CheckProjectRole 적용
```

### 확인 및 테스트 프롬프트

```
Phase 6 AOP 구현이 올바르게 동작하는지 확인해줘:

1. 빌드 성공: './gradlew build -x test'
2. 테스트 시나리오:
   a. memberA(LEADER)로 이슈 생성 → 성공
   b. memberB(VIEWER)로 이슈 생성 → 403
   c. memberA로 이슈 상태 변경 → issue_histories에 이력 자동 기록 확인
   d. memberA로 이슈 제목+우선순위 변경 → 2개 이력 기록 확인
3. AOP 실행 순서 확인: 권한 체크 → 비즈니스 로직 → 이력 저장
```

---

# Phase 7: QueryDSL + N+1 문제 해결

## 이 Phase에서 추가하는 의존성

| 라이브러리 | 용도 |
|-----------|------|
| `querydsl-jpa:5.1.0:jakarta` | QueryDSL JPA 지원 (Jakarta 버전) |
| `querydsl-apt:5.1.0:jakarta` | 컴파일 시 Q 클래스 자동 생성 (APT) |

### build.gradle에 추가할 내용

```groovy
dependencies {
    // === Phase 1 ~ 5 (기존) === //
    // ... (생략) ...

    // === Phase 7: QueryDSL === //
    implementation 'com.querydsl:querydsl-jpa:5.1.0:jakarta'
    annotationProcessor 'com.querydsl:querydsl-apt:5.1.0:jakarta'
    annotationProcessor 'jakarta.annotation:jakarta.annotation-api'
    annotationProcessor 'jakarta.persistence:jakarta.persistence-api'
}
```

> **왜 이제 QueryDSL을 추가하는가?** 이슈 검색에 동적 조건(상태, 타입, 우선순위, 담당자, 키워드 등)이 필요한데, JPQL 문자열로는 복잡한 동적 쿼리를 작성하기 어렵다. QueryDSL은 Java 코드로 타입 세이프한 쿼리를 작성할 수 있게 해준다.

## 7.1 N+1 문제 상세 분석

> **이 섹션이 Phase 7에 있는 이유:** Phase 3에서 N+1 개념을 소개했지만, 실제 해결은 QueryDSL + Fetch Join을 함께 사용하는 이 Phase에서 한다.

### N+1 문제란?

```java
List<Issue> issues = issueRepository.findAllByProjectId(projectId);
// SQL 1번: SELECT * FROM issues WHERE project_id = 1 → 100건

for (Issue issue : issues) {
    issue.getAssignee().getName(); // 각 이슈마다 SQL 1번씩 추가!
    issue.getReporter().getName(); // 또 SQL 1번씩!
}
// 총: 1 + 100 + 100 = 201번의 SQL!
```

### 해결 방법 1: Fetch Join (JPQL)

```java
@Query("SELECT i FROM Issue i " +
       "JOIN FETCH i.assignee " +
       "JOIN FETCH i.reporter " +
       "JOIN FETCH i.project " +
       "WHERE i.project.id = :projectId")
List<Issue> findAllByProjectIdWithDetails(@Param("projectId") Long projectId);
```

### 해결 방법 2: @BatchSize (OneToMany N+1 해결)

```java
@OneToMany(mappedBy = "issue")
@BatchSize(size = 100) // IN 절로 묶어서 조회
private List<Comment> comments = new ArrayList<>();
```

## 7.2 QueryDSL 딥다이브

### Q 클래스란?

QueryDSL은 APT를 사용하여 엔티티에서 Q 클래스를 자동 생성한다:
- `@Entity Issue` → `QIssue`
- `./gradlew compileJava` 실행 시 생성됨

### Custom Repository 패턴

```
IssueRepository
├── extends JpaRepository<Issue, Long>      ← 기본 CRUD
└── extends IssueRepositoryCustom            ← QueryDSL 커스텀 쿼리

IssueRepositoryCustom (인터페이스)
IssueRepositoryImpl (구현 클래스, 이름 규칙: Impl 접미사!)
```

### 동적 조건 검색 구현

```java
// null이면 조건 무시
private BooleanExpression statusEq(IssueStatus status) {
    return status != null ? issue.status.eq(status) : null;
}
```

## 7.3 Phase 7 Claude Code 구현 프롬프트

### 구현 프롬프트

```
TaskSyncBoard Phase 7: QueryDSL + N+1 문제 해결을 구현해줘.

★ build.gradle에 QueryDSL 의존성 + APT 설정을 추가해줘.

1. QueryDslConfig: JPAQueryFactory Bean 등록
2. './gradlew compileJava' 실행하여 Q 클래스 생성 확인
3. IssueSearchCondition DTO
4. IssueRepositoryCustom + IssueRepositoryImpl (QueryDSL 동적 검색)
5. Fetch Join 메서드 추가 (findByIdWithDetails)
6. Issue 엔티티 comments에 @BatchSize(size = 100) 추가
7. IssueController에 검색/통계 엔드포인트 추가
```

### 확인 및 테스트 프롬프트

```
Phase 7 QueryDSL 구현이 올바른지 확인해줘:

1. Q 클래스 생성 확인
2. 빌드 성공
3. 동적 검색 테스트 (다양한 조건 조합)
4. N+1 확인: show-sql에서 SQL 실행 횟수 확인
```

---

# Phase 8: 동시성 제어 (낙관적 락 / 비관적 락)

## 이 Phase에서 추가하는 의존성

**없음.** JPA의 `@Version`으로 낙관적 락을 구현한다. 이미 Phase 3에서 Issue 엔티티에 `@Version` 필드를 추가해 두었다.

## 8.1 동시성 문제란?

두 작업자가 동시에 같은 이슈를 수정하면 한 쪽의 변경이 덮어씌워진다 (Lost Update).

## 8.2 낙관적 락 (Optimistic Locking)

```java
@Entity
public class Issue {
    @Version  // ← 이것 하나로 낙관적 락 활성화!
    private Long version;
}
```

- UPDATE 시 `WHERE version = ?` 조건이 자동 추가
- 다른 사용자가 먼저 수정했으면 `ObjectOptimisticLockingFailureException` 발생
- 클라이언트는 Request에 version을 포함하고, 409 Conflict 시 최신 데이터를 다시 조회

## 8.3 비관적 락 (Pessimistic Locking) - 학습용

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)  // SELECT ... FOR UPDATE
@Query("SELECT i FROM Issue i WHERE i.id = :id")
Optional<Issue> findByIdWithPessimisticLock(@Param("id") Long id);
```

> **TaskSyncBoard에서는 낙관적 락을 사용한다.** 이슈 수정은 충돌 빈도가 낮고, 사용자에게 알리고 재시도하게 하는 것이 적절하다.

## 8.4 Phase 8 Claude Code 구현 프롬프트

### 구현 프롬프트

```
TaskSyncBoard Phase 8: 동시성 제어 (낙관적 락)를 구현해줘.

(build.gradle 수정 불필요)

1. Issue 엔티티에 @Version 필드 확인
2. IssueUpdateRequest, IssueStatusChangeRequest에 version 필드 추가
3. IssueResponse에 version 필드 포함
4. GlobalExceptionHandler에 OptimisticLockingFailureException 핸들러 확인
5. 비관적 락 예시 메서드 하나 추가 (학습용, 주석으로 설명)
```

### 확인 및 테스트 프롬프트

```
Phase 8 동시성 제어 확인:

1. 이슈 생성 → version=0 확인
2. 수정 (version=0) → 성공, version=1
3. 같은 이슈 수정 (version=0, 옛날 버전) → 409 Conflict
```

---

# Phase 9: 테스트 전략 (JUnit 5 + Mockito + Testcontainers)

## 이 Phase에서 추가하는 의존성

| 라이브러리 | 용도 |
|-----------|------|
| `org.testcontainers:testcontainers` | Testcontainers 코어 |
| `org.testcontainers:postgresql` | PostgreSQL Testcontainer 모듈 |
| `org.testcontainers:junit-jupiter` | JUnit 5 통합 |

### build.gradle에 추가할 내용

```groovy
dependencies {
    // === Phase 1 ~ 7 (기존) === //
    // ... (생략) ...

    // === Phase 9: Testcontainers === //
    testImplementation 'org.testcontainers:testcontainers'
    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'org.testcontainers:junit-jupiter'
}
```

> **왜 이제 Testcontainers를 추가하는가?** 모든 기능이 완성되었으니, 이제 단위/통합 테스트로 검증한다. Testcontainers는 **프로덕션과 동일한 PostgreSQL + Redis**를 테스트에서 Docker로 자동 띄워준다.

> **참고:** JUnit 5, Mockito, AssertJ는 Phase 1에서 추가한 `spring-boot-starter-test`에 이미 포함되어 있다. `spring-security-test`는 Phase 5에서 추가했다.

## 9.1 테스트 피라미드

```
         ╱╲
        ╱  ╲         E2E 테스트 (느림, 비쌈, 적게)
       ╱────╲
      ╱      ╲       통합 테스트 (중간) → Testcontainers
     ╱────────╲
    ╱          ╲
   ╱────────────╲    단위 테스트 (빠름, 싸름, 많이) → Mockito
  ╱              ╲
 ╱________________╲
```

## 9.2 Testcontainers 딥다이브

### 왜 필요한가?

H2 같은 인메모리 DB로 테스트하면 PostgreSQL과 문법/동작이 달라 "테스트 통과 → 프로덕션 실패"가 발생할 수 있다. Testcontainers는 **Docker 컨테이너를 자동으로 띄우고 테스트가 끝나면 자동으로 내린다.**

### 테스트 기반 설정

```java
@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
public abstract class IntegrationTestSupport {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("tasksyncboard_test")
            .withUsername("test_user")
            .withPassword("test_password");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }
}
```

## 9.3 Phase 9 Claude Code 구현 프롬프트

### 구현 프롬프트

```
TaskSyncBoard Phase 9: 테스트 코드를 구현해줘.

★ build.gradle에 Testcontainers 의존성을 추가해줘.

1. 테스트 기반 클래스: IntegrationTestSupport, RepositoryTestSupport, MockMvcTestSupport
2. application-test.yml
3. 단위 테스트 (Mockito): IssueServiceTest, MemberServiceTest, AuthServiceTest
4. 통합 테스트 (Testcontainers): IssueIntegrationTest, AuthIntegrationTest
5. Repository 테스트: IssueRepositoryTest (QueryDSL 검증)
6. './gradlew test' → 전체 통과 확인
```

---

# Phase 10: Swagger (Springdoc OpenAPI) 문서화

## 이 Phase에서 추가하는 의존성

| 라이브러리 | 용도 |
|-----------|------|
| `springdoc-openapi-starter-webmvc-ui:2.6.0` | Swagger UI + OpenAPI 3.0 자동 문서 생성 |

### build.gradle에 추가할 내용

```groovy
dependencies {
    // === Phase 1 ~ 9 (기존) === //
    // ... (생략) ...

    // === Phase 10: Swagger === //
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0'
}
```

> **왜 이제 Swagger를 추가하는가?** 모든 API가 완성되고 테스트도 통과했으니, 프론트엔드 개발자나 QA 팀이 사용할 API 문서를 자동 생성한다.

## 10.1 설정 및 사용법

```java
@Configuration
public class SwaggerConfig {

    @Bean
    public OpenAPI openAPI() {
        String jwtSchemeName = "JWT Authentication";
        SecurityRequirement securityRequirement = new SecurityRequirement().addList(jwtSchemeName);

        SecurityScheme securityScheme = new SecurityScheme()
                .name(jwtSchemeName)
                .type(SecurityScheme.Type.HTTP)
                .scheme("bearer")
                .bearerFormat("JWT");

        return new OpenAPI()
                .info(new Info()
                        .title("TaskSyncBoard API")
                        .description("사내 협업/이슈 관리 시스템 API")
                        .version("v1.0.0"))
                .addSecurityItem(securityRequirement)
                .components(new Components().addSecuritySchemes(jwtSchemeName, securityScheme));
    }
}
```

접속: `http://localhost:8080/swagger-ui.html`

## 10.2 Phase 10 Claude Code 구현 프롬프트

### 구현 프롬프트

```
TaskSyncBoard Phase 10: Swagger 문서화를 구현해줘.

★ build.gradle에 springdoc-openapi-starter-webmvc-ui:2.6.0 의존성을 추가해줘.

1. SwaggerConfig: OpenAPI Bean + JWT 인증 스키마
2. 모든 Controller에 @Tag, @Operation, @ApiResponses 추가
3. Request DTO에 @Schema 어노테이션 추가
4. application.yml에 springdoc 설정 추가
5. SecurityConfig에서 /swagger-ui/**, /v3/api-docs/** permitAll 확인
```

---

# Phase 11: Actuator 모니터링 + Docker 이미지 빌드

## 이 Phase에서 추가하는 의존성

| 라이브러리 | 용도 |
|-----------|------|
| `spring-boot-starter-actuator` | 애플리케이션 내부 상태 모니터링 엔드포인트 |

### build.gradle에 추가할 내용 (최종 완성)

```groovy
dependencies {
    // === Phase 1: 프로젝트 뼈대 === //
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    // === Phase 3: JPA + DB === //
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql'
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql'

    // === Phase 4: Validation === //
    implementation 'org.springframework.boot:spring-boot-starter-validation'

    // === Phase 5: Security + JWT + Redis === //
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'io.jsonwebtoken:jjwt-api:0.12.6'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.6'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.6'
    testImplementation 'org.springframework.security:spring-security-test'

    // === Phase 7: QueryDSL === //
    implementation 'com.querydsl:querydsl-jpa:5.1.0:jakarta'
    annotationProcessor 'com.querydsl:querydsl-apt:5.1.0:jakarta'
    annotationProcessor 'jakarta.annotation:jakarta.annotation-api'
    annotationProcessor 'jakarta.persistence:jakarta.persistence-api'

    // === Phase 9: Testcontainers === //
    testImplementation 'org.testcontainers:testcontainers'
    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'org.testcontainers:junit-jupiter'

    // === Phase 10: Swagger === //
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0'

    // === Phase 11: Actuator === //
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

> **이것이 최종 build.gradle이다.** Phase 1에서 2개 의존성으로 시작하여, 11개의 Phase를 거치며 점진적으로 성장했다.

## 11.1 Spring Boot Actuator

Actuator는 애플리케이션의 **내부 상태를 모니터링**하는 엔드포인트를 제공한다.

### 주요 엔드포인트

| 엔드포인트 | 설명 |
|-----------|------|
| `/actuator/health` | 애플리케이션 + DB + Redis 연결 상태 |
| `/actuator/info` | 애플리케이션 정보 |
| `/actuator/metrics` | JVM 메모리, CPU, HTTP 요청 수 등 메트릭 |
| `/actuator/flyway` | Flyway 마이그레이션 이력 |

### 설정

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, flyway
  endpoint:
    health:
      show-details: when-authorized
```

## 11.2 Docker 이미지 빌드 (멀티 스테이지)

```dockerfile
# === Stage 1: Build === #
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
COPY build.gradle settings.gradle gradlew ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon || true
COPY src ./src
RUN ./gradlew bootJar --no-daemon -x test

# === Stage 2: Runtime === #
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "-jar", "app.jar"]
```

### docker-compose에 앱 추가 (최종)

```yaml
version: '3.8'

services:
  app:
    build:
      context: ..
      dockerfile: Dockerfile
    container_name: tsb-app
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: local
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/tasksyncboard
      SPRING_DATASOURCE_USERNAME: tsb_user
      SPRING_DATASOURCE_PASSWORD: tsb_password
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PORT: 6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    container_name: tsb-postgres
    environment:
      POSTGRES_DB: tasksyncboard
      POSTGRES_USER: tsb_user
      POSTGRES_PASSWORD: tsb_password
    ports:
      - "5432:5432"
    volumes:
      - tsb-postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tsb_user -d tasksyncboard"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: tsb-redis
    ports:
      - "6379:6379"
    volumes:
      - tsb-redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  tsb-postgres-data:
  tsb-redis-data:
```

## 11.3 Phase 11 Claude Code 구현 프롬프트

### 구현 프롬프트

```
TaskSyncBoard Phase 11: Actuator + Docker 이미지 빌드를 구현해줘.

★ build.gradle에 spring-boot-starter-actuator 의존성을 추가해줘.

1. Actuator 설정 (application.yml)
2. SecurityConfig에서 /actuator/health permitAll, /actuator/** ADMIN만
3. Dockerfile 생성 (멀티 스테이지 빌드)
4. docker-compose.yml 업데이트 (app 서비스 추가)
5. .dockerignore 파일 생성
```

### 확인 및 테스트 프롬프트

```
Phase 11 최종 패키징 확인:

1. GET /actuator/health → {"status":"UP"}
2. 'docker build -t tasksyncboard:latest .' 성공
3. 'docker-compose -f docker/docker-compose.yml up --build' 전체 스택 실행
4. API 호출 + Swagger UI 접근 확인
```

---

# 부록 A: 커스텀 로깅 전략

## A.1 API 요청/응답 로깅 (AOP)

```java
@Slf4j
@Aspect
@Component
public class ApiLoggingAspect {

    @Around("within(@org.springframework.web.bind.annotation.RestController *)")
    public Object logApiCall(ProceedingJoinPoint joinPoint) throws Throwable {
        String className = joinPoint.getTarget().getClass().getSimpleName();
        String methodName = joinPoint.getSignature().getName();

        log.info("[API 요청] {}.{} | args={}", className, methodName,
                Arrays.toString(joinPoint.getArgs()));

        long startTime = System.currentTimeMillis();

        try {
            Object result = joinPoint.proceed();
            long elapsed = System.currentTimeMillis() - startTime;
            log.info("[API 응답] {}.{} | {}ms", className, methodName, elapsed);
            return result;
        } catch (Exception e) {
            long elapsed = System.currentTimeMillis() - startTime;
            log.error("[API 에러] {}.{} | {}ms | error={}",
                    className, methodName, elapsed, e.getMessage());
            throw e;
        }
    }
}
```

## A.2 Logback 설정 (logback-spring.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProfile name="local">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <springProfile name="prod">
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/tasksyncboard.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>logs/tasksyncboard.%d{yyyy-MM-dd}.log</fileNamePattern>
                <maxHistory>30</maxHistory>
            </rollingPolicy>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>
</configuration>
```

---

# 부록 B: ERD 및 도메인 상세 설계

## B.1 전체 ERD (텍스트)

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│   members    │     │ project_members   │     │  projects    │
├──────────────┤     ├──────────────────┤     ├──────────────┤
│ member_id PK │◄───┤ member_id FK     │     │ project_id PK│
│ email        │     │ project_id FK    │────►│ name         │
│ password     │     │ project_role     │     │ description  │
│ name         │     │ joined_at        │     │ project_key  │
│ role         │     └──────────────────┘     │ status       │
│ status       │                               └──────┬───────┘
│ created_at   │                                      │
│ updated_at   │◄──────────────────────────────┐      │
└──────────────┘                                │      │
       ▲                                        │      │
       │ reporter_id                            │      │
       │ assignee_id                            │      │
┌──────┴───────┐     ┌──────────────────┐      │      │
│   issues     │     │ issue_histories   │      │      │
├──────────────┤     ├──────────────────┤      │      │
│ issue_id PK  │◄───┤ issue_id FK      │      │      │
│ title        │     │ changed_by FK    │──────┘      │
│ description  │     │ field_name       │             │
│ type         │     │ old_value        │             │
│ priority     │     │ new_value        │             │
│ status       │     │ changed_at       │             │
│ project_id FK│─────┼─────────────────────────────────┘
│ reporter_id  │     └──────────────────┘
│ assignee_id  │
│ version      │     ┌──────────────────┐
│ created_at   │     │  comments        │
│ updated_at   │     ├──────────────────┤
└──────┬───────┘     │ comment_id PK    │
       │             │ content          │
       │             │ issue_id FK      │────► issues
       │             │ author_id FK     │────► members
       │             │ created_at       │
       │             │ updated_at       │
       │             └──────────────────┘
       │
       │  ┌──────────────┐     ┌──────────────┐
       └──┤ issue_labels  │────►│   labels     │
          ├──────────────┤     ├──────────────┤
          │ issue_id FK  │     │ label_id PK  │
          │ label_id FK  │     │ name         │
          └──────────────┘     │ color        │
                               │ project_id FK│
                               └──────────────┘
```

## B.2 상태 전이 다이어그램 (Issue)

```
               ┌─────────────────┐
               │                 │
       ┌──────►│      TODO       │◄──────┐
       │       │                 │       │
       │       └────────┬────────┘       │
       │                │                │
       │                ▼                │
       │       ┌─────────────────┐       │
       │       │                 │       │
       │       │  IN_PROGRESS    │───────┘ (재오픈)
       │       │                 │
       │       └────────┬────────┘
       │                │
       │                ▼
       │       ┌─────────────────┐
       │       │                 │
       │       │   IN_REVIEW     │
       │       │                 │
       │       └────────┬────────┘
       │                │
       │                ▼
       │       ┌─────────────────┐
       │       │                 │
       │       │      DONE       │
       │       │                 │
       │       └─────────────────┘
       │
       │       ┌─────────────────┐
       │       │                 │
       └───────│   CANCELED      │  (어떤 상태에서든 취소 가능)
               │                 │
               └─────────────────┘
```

---

# 최종 체크리스트

모든 Phase를 완료한 후 아래를 확인한다:

- [ ] `./gradlew build` → 빌드 성공 (컴파일 + 테스트 통과)
- [ ] `docker-compose up --build` → 전체 스택 실행 성공
- [ ] Swagger UI (`/swagger-ui.html`) → 모든 API 표시
- [ ] 회원가입 → 로그인 → JWT 발급 → API 호출 → 정상 동작
- [ ] 이슈 생성/수정/상태변경 → AOP 이력 자동 기록
- [ ] 권한 체크 → VIEWER가 이슈 생성 시도 → 403 거부
- [ ] 동시 수정 → 낙관적 락 → 409 Conflict
- [ ] Actuator `/health` → DB + Redis 연결 상태 확인
- [ ] `./gradlew test` → 단위 + 통합 테스트 전체 통과

---

## 의존성 추가 이력 요약

| Phase | 추가한 의존성 | 누적 의존성 수 |
|-------|-------------|--------------|
| Phase 1 | `spring-boot-starter-web`, `lombok` | 2 |
| Phase 2 | (없음 - Docker만) | 2 |
| Phase 3 | `spring-boot-starter-data-jpa`, `postgresql`, `flyway-core`, `flyway-database-postgresql` | 6 |
| Phase 4 | `spring-boot-starter-validation` | 7 |
| Phase 5 | `spring-boot-starter-security`, `spring-boot-starter-data-redis`, `jjwt-api`, `jjwt-impl`, `jjwt-jackson`, `spring-security-test` | 13 |
| Phase 6 | (없음 - AOP는 이미 포함) | 13 |
| Phase 7 | `querydsl-jpa`, `querydsl-apt` | 15 |
| Phase 8 | (없음 - JPA @Version 사용) | 15 |
| Phase 9 | `testcontainers`, `testcontainers-postgresql`, `testcontainers-junit-jupiter` | 18 |
| Phase 10 | `springdoc-openapi-starter-webmvc-ui` | 19 |
| Phase 11 | `spring-boot-starter-actuator` | 20 |
