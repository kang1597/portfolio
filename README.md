# 강윤아 · Java / Spring 백엔드 개발자

> eGovFrame(Spring MVC · MyBatis · JSP) 기반으로 **공공기관·협회 웹 플랫폼의 신규 구축과 유지보수**를 해 왔습니다.
> 요구사항 분석부터 DB 설계, 구현, 테스트, 배포·운영, 보안 점검 대응까지 전 과정을 경험했고,
> 실무에서 겪은 문제를 개인 프로젝트로 가져와 **Spring Boot · JPA · Redis로 다시 풀고, 동시성 문제를 직접 재현해 측정·비교**했습니다.
> 반복 업무를 도구로 자동화하고 문제의 근본 원인을 끝까지 찾는 것을 중요하게 생각합니다.

- Email : 35wkd2817@naver.com
- GitHub : [content-ranking-system](https://github.com/kang1597/content-ranking-system) · [db-metadata-generator](https://github.com/kang1597/db-metadata-generator)

<br>

## 개인 프로젝트

### 1. Content Ranking System — 콘텐츠 좋아요/랭킹 집계 시스템 · [GitHub](https://github.com/kang1597/content-ranking-system)

`2026.09 ~ 2026.11` · 개인 프로젝트 (설계 · 구현 · 측정 전체)
`Java 17` `Spring Boot 4` `Spring Security 7` `JPA(Hibernate 7)` `MySQL 8` `Redis 7` `Docker Compose` `GitHub Actions`

좋아요 수를 비정규화 컬럼에 저장하면서 발생하는 **갱신 손실(Lost Update)** 을 동시 요청 100건으로 재현하고,
**6가지 동시성 제어 방식을 동일 조건에서 측정해 비교**했습니다.

**측정 결과** (요청 100건 / 스레드 32 / 커넥션 풀 10)

| 전략 | 처리량 | 정합성 | 초당 처리 |
| --- | --- | --- | --- |
| 조회 후 UPDATE | 12% | ✅ | 5.2건/s |
| 순서만 변경 | 10% | ✅ | 9.8건/s |
| 비관적 락 | 100% | ✅ | 22.0건/s |
| 낙관적 락 | 100% | ✅ | 15.5건/s |
| **DB 원자적 연산** | **100%** | **✅** | **45.0건/s** |
| Redis 원자적 연산 | 100% | ✅(Redis) | 234건/s |

- **DB 원자적 연산 채택** — 값을 읽지 않고 DB에 연산을 맡겨 읽고-쓰는 틈을 제거. 락 보유 구간이 가장 짧아 비관적 락의 2배 처리량. Redis 방식이 7배 빨랐으나 *진실이 휘발성 저장소에 있으면 안 된다*고 판단해 제외하고, 전환 임계점을 문서화
- **FK 공유 락으로 인한 데드락 발견 및 해결** — INSERT가 부모 행에 거는 S 락과 뒤이은 UPDATE의 X 락이 순환 대기를 일으켜 100건 중 80건 실패. 락 획득 순서를 바꿔 제거
- **조용한 실패 발견** — 데드락만 없앤 구현이 100% 성공을 보고하면서 90%를 유실. `@Version`으로 조용한 유실을 명시적 실패로 전환
- **Redis 캐싱** — 좋아요 수는 Cache-Aside(읽기 캐시), 조회수는 증분 누적 + 배치 동기화(쓰기 버퍼)로 **요구 정확도에 따라 다르게 설계**
- **랭킹(Sorted Set) 유실 복구** — 임시 키 + `RENAME`으로 무중단 원자적 교체, 조회 시 지연 재구축. `FLUSHDB` 후 자동 복구 검증(450ms → 30ms)
- **테스트 격리 / CI** — 전용 MySQL(tmpfs) + Redis 논리 DB 분리, GitHub Actions service container로 자동 검증

<br>

### 2. DB Metadata Generator — 테이블 정의서 · VO 자동 생성 도구 · [GitHub](https://github.com/kang1597/db-metadata-generator)

`2026.08 ~ 2026.11` · 개인 프로젝트 (설계 · 구현 · 배포 전체)
`Java 17` `Spring Boot` `Spring Data JPA` `JDBC` `Gradle` `PostgreSQL 16` `MariaDB 11` `Apache POI` `Freemarker` `Jasypt` `JUnit 5` `AssertJ` `Docker` `GitHub Actions`

실무에서 테이블 정의서를 수기로 관리하다 보니 정의서와 실제 스키마가 계속 어긋났습니다.
회사에서 만들었던 정의서 자동화 도구의 아이디어를 개인 프로젝트로 다시 설계하고, 레거시 스택(eGovFrame · MyBatis · XML 설정)을 Spring Boot · JPA · 어노테이션 기반으로 전환해 구현했습니다.

- 등록한 DB의 메타데이터(테이블·컬럼·타입·길이·NULL·PK·코멘트)를 JDBC DatabaseMetaData로 추출해 **Excel 정의서와 Lombok 기반 Java VO 자동 생성** (REST API 8종)
  - Excel은 서버에 임시 파일을 남기지 않고 메모리에서 바로 내려받기, VO는 snake_case → camelCase·DB → Java 타입 변환과 필요한 import만 생성
- 고정 스키마인 애플리케이션 DB는 JPA, 런타임에 동적으로 연결하는 대상 DB는 JDBC로 분리한 **이중 DB 접근 구조** 설계 (기능 단위 패키지 구성)
- DB 접속 비밀번호를 JPA AttributeConverter + Jasypt(AES-256)로 저장·조회 시 자동 암·복호화, 암호화 키는 환경변수로 주입
- **트러블슈팅 7건을 README에 기록**
  - MariaDB 조회 시 시스템 테이블이 섞이는 문제 → DB별 catalog/schema 개념 차이로 규명, 현재 catalog로 조회 범위를 제한하고 회귀 테스트 추가
  - Jasypt AES 암호화 실패 → AES 계열에 필요한 IV 생성기 설정 누락이 원인
  - 컨테이너화 후 `localhost` 접속 실패 → Compose 서비스명으로 접속하도록 정리
  - 한글 파일명 다운로드 깨짐 → `filename*=UTF-8''` 형식으로 인코딩
- 테스트 13개(단위 7 · PostgreSQL/MariaDB 통합 6)를 GitHub Actions에서 push마다 두 DB 컨테이너로 자동 실행
- Docker 멀티 스테이지 빌드와 의존성 레이어 캐싱으로 실행 이미지 **176MB**, Docker Compose로 앱·DB 일괄 실행

<br>

## 경력 요약 (총 2년 3개월)

| 회사 | 기간 | 소속 · 직급 | 주요 업무 |
|---|---|---|---|
| **나우리** | 2025.10 ~ 재직 중 (1년 2개월) | AI사업부 · 사원 | 공공기관·협회 웹 시스템 백엔드 개발 및 유지보수, 보안 취약점 조치, 개발 서버·CI/CD 구축 |
| **나라원시스템 (구 아사달시스템)** | 2023.03 ~ 2024.03 (1년 1개월) | 시스템사업부 · 사원 | 공공기관 본인확인 지원포털 유지보수 (시큐어코딩 · DB 암호화 · 웹 접근성 · 웹 취약점 · 서버 장애 대응 · 백업 자동화) |

<br>

## 프로젝트 경력 — 나우리

### 1. MAINBiz 상생플랫폼 구축 ⭐ 대표 프로젝트

`2026.02 ~ 현재` · 백엔드 개발 (요구분석 · 설계 · 구현 · 테스트 · 배포)
`eGovFrame 4.3` `Spring MVC` `MyBatis` `MariaDB` `JSP + Tiles` `CLIP Report 5.0` `Linux`

중소기업 협회 회원사 5천여 곳이 쓰는 멀티사이트 플랫폼(본회 + 연합회 11 + 지회 96)을 구축하고 운영 전환을 준비했습니다.

- **회원사 공동구매(B2B 마켓) 모듈 신규 구축**: 등록 → 승인 → 모집 → 신청 → 거래결과 흐름 설계, 상태 자동 전환 배치, 연장·조기마감·취소 승인, 마이페이지 4탭, 현황통계, 알림 9종
- **보안 점검 대응**: 보안성 테스팅 담당 항목 조치(인증·권한·파일 업로드 영역), 정적 분석(SAST) 결과 분류·조치와 화면 전반 회귀 테스트
- **데이터 이관 · 성능**: 협회비 납부내역 **29,179건** 등 레거시 데이터 이관, 회비 갱신관리 조회 **30초~1분 → 0.035초**
- **멀티사이트 구조 정비**: 107개 사이트의 중복 게시판을 공용 게시판 4개로 통합(스키마 변경 없이 사이트별 데이터 격리)
- **행사 · 증명서 · 알림**: 행사 신청·승인·출석 스케줄러, 서면의결서·위임장 등 CLIP Report 증명서, CompletableFuture 병렬 메일 발송
- **운영 전환 준비**: 회원등급별 전수 테스트 자동화, 교육 매뉴얼(사용자 100장·운영자 117장)과 프로그램목록 작성

👉 **[MAINBiz 상세 보기 (주요 성과 · 트러블슈팅 · 작업 타임라인)](./MAINBiz.md)**

<br>

### 2. JDCC 시스템 유지보수 및 기능 개발

`2025.10 ~ 2026.03` · 백엔드 개발 및 유지보수
`eGovFrame 4.3.1` `Spring MVC` `MyBatis` `JSP/JSTL` `Tiles` `MariaDB` `Oracle` `Linux` `Tomcat` `Jenkins`

- 기존 구조(Controller → Service → DAO/Mapper)를 분석하고 공통 게시판·페이징을 재사용해 문의·공지·게시판 기능 개발
- **개인정보처리방침 관리** 기능 신규 개발 (관리자 목록·상세·등록·삭제, 버전 순번), 회원가입 약관 사용자 화면 표출
- 사용자별 `내 문의내역` 조회 기능 개발 (메인 · 마이페이지)
- **웹 취약점 일괄 조치**
  - 출력 이스케이프용 JSTL 커스텀 함수를 만들어 화면 전반에 적용
  - 조회 요청 메서드 GET → POST 전환, 입력값 깨짐·출력 처리 보완
- 회원정보 등록·수정 시 **RSA 기반 개인정보 전송**, DB 암호화, 중요 정보 변경 시 이중 확인 절차 적용
- SSO · KSign 인증 모듈의 라이선스·연동 오류 원인 분석 및 조치
- MariaDB 개발 DB 구성, Rocky Linux · Tomcat 운영, Jenkins · SVN 기반 빌드·배포 직접 수행

<br>

### 3. MAINBiz 개발 서버 인프라 · CI/CD 구축

`2026.01` · 서버 구축 및 배포 자동화 (단독 수행)
`Rocky Linux 9.6` `Jenkins` `SVN` `Tomcat` `SSH` `systemd`

- MAINBiz 프로젝트의 Rocky Linux 9.6 개발 서버 구축 및 SVN 저장소 구성
- Jenkins 기반 **체크아웃 → 빌드 → 배포 파이프라인** 구축, SSH Key 인증으로 자동 배포
- Tomcat을 systemd 서비스로 등록해 서버 재기동 시 자동 실행되도록 구성

<br>

### 4. 공공기관 CMS 웹사이트 개발 및 유지보수

`2025.10 ~ 현재` · 개발 및 유지보수
`eGovFrame` `Java` `JavaScript` `Oracle` `Linux`

- 약관 · 개인정보처리방침 관리 기능 개발 및 화면 기능 개선
- XSS · SQL Injection 및 웹 접근성 점검 결과 조치, 개인정보 암호화 적용
- Linux crontab 기반 서버 백업 자동화, 운영 중 장애·데이터 문제 원인 분석 및 조치

<br>

### 5. 협력업체 관리보고서 · 체크리스트 시스템

`2025.12` · 기능 설계·개발
`Spring` `MyBatis` `Kendo UI`

- 감사 담당 부서의 실제 업무 순서를 먼저 파악해 설계
- 업체·항목별 **동적 체크리스트**, 임시저장, 감사 결과·증빙자료 관리 기능 개발
- 여러 업체의 관리보고서와 증빙사진을 포함한 **PDF 일괄 생성** 기능 구현

<br>

### 6. 전주시청 빅데이터포털 유지보수

`2026.02 ~ 현재` · SNS 수집 크롤러 단독 운영
`Python` `Selenium` `Linux` `JSP`

- 대상 사이트의 구조 변경에 맞춘 파싱 로직 수정, 포털 화면(JSP) 차트·데이터 표시 오류 디버깅
- 장기 미종료(좀비) 프로세스로 수집이 중단되는 장애 진단·조치, IP 차단 발생 시 원인 분석

<br>

## 프로젝트 경력 — 나라원시스템 (구 아사달시스템)

### 1. 본인확인 지원포털 시스템 유지보수

`2023.03 ~ 2024.03` · 웹개발자 (유지보수)
`eGovFrame` `Java` `JSP` `JavaScript` `Oracle` `MySQL` `Linux` `crontab`

- 전자정부프레임워크 기반 공공기관 본인확인 지원포털의 기능 개선과 운영 유지보수
- **시큐어코딩** 기준에 따른 소스 보완과 **DB 정보 암호화**, **웹 취약점** 점검 결과 조치
- **웹 접근성** 점검 항목 대응 및 화면 개선
- 서버 디스크 용량 부족으로 서비스가 중단된 장애의 원인을 확인하고 복구·재발 방지 조치
- Linux **crontab 기반 백업 파일·서버 용량 관리 자동화**
- 기존 UI와 충돌하는 동영상 페이지 요구사항을 퍼블리셔와 협의해 대안(기관 유튜브 링크 활용)을 제안, 일정 내 해결

<br>

### 2. CMS 활용 웹툰 게시판 (사내 자체 프로젝트)

`Java` `JSP` `사내 CMS`

- 게시글 CRUD와 **좋아요 · 조회수 · 조회수 기반 순위** 기능 구현 (같은 기능을 개인 프로젝트 [Content Ranking System](https://github.com/kang1597/content-ranking-system)에서 동시성 관점으로 다시 다룸)

<br>

## 기술 스택

| 구분 | 사용 기술 |
|---|---|
| Backend | Java 17, eGovFrame 4.x, Spring MVC, MyBatis, Lombok · (개인) Spring Boot, Spring Data JPA, Gradle |
| Frontend | JSP / JSTL, Tiles, jQuery, Kendo UI, CKEditor, ECharts |
| DB · Cache | MariaDB, Oracle, PostgreSQL (실행계획 분석 · 인덱스 튜닝) · (개인) MySQL, Redis |
| Infra / DevOps | Rocky Linux, Tomcat, Jenkins, SVN, Git · (개인) Docker, GitHub Actions |
| Report / File | CLIP Report 5.0, Apache POI, Apache PDFBox |
| 보안 | 주요정보통신기반시설 기준 보안성 테스팅 대응, Sparrow SAST, 시큐어코딩(개인정보 암호화 · XSS · 권한 검증) |
| 자동화 · 테스트 | Python(Selenium, Playwright), JUnit 5, Claude Code(AI 코딩 도구) |

<br>

## 교육 · 자격

| 구분 | 내용 | 기간 |
|---|---|---|
| 학력 | 원광대학교 컴퓨터소프트웨어공학과 졸업 (3.9 / 4.5) | 2018.03 ~ 2023.02 |
| 교육 | 전주정보문화산업진흥원 · 디지털(AI) 융합기반 웹&앱 전문개발자 양성과정 | 2025.06 ~ 2025.09 |
| 교육 | 멀티캠퍼스 · [멀티잇] 채용연계 풀스택 개발자 취업캠프(Java) | 2022.09 ~ 2023.03 |
| 자격증 | SQL 개발자 (SQLD) · 한국데이터산업진흥원 | 2024.09 |

<br>

---

※ 회사·고객사 소스코드와 데이터는 포함하지 않았으며, 내부 정보(서버 주소·계정·회원 정보·보안 점검 세부 내용)는 싣지 않았습니다.
