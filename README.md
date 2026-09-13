# Korea Pulse Backend

> 지금 대한민국이 무엇에 주목하고 있는지 한눈에

국내 뉴스·검색 데이터를 분석해 **카테고리별 급상승 이슈 TOP 10**과 **이슈 상세(추이·관련 뉴스·AI 요약)** 를 제공하는 Korea Pulse의 백엔드 서버입니다.

## 기술 스택

| 구분 | 내용 |
| --- | --- |
| 언어 / 프레임워크 | Java 21, Spring Boot 4.1.1 |
| DB | PostgreSQL (로컬 개발: H2) |
| ORM | Spring Data JPA |
| 배포 | Railway |
| 주요 외부 연동 | 네이버 뉴스 검색 API, 네이버 데이터랩, Google Trends RSS, LLM 요약 API |

## 동작 개요

- 스케줄러가 30분 주기로 파이프라인을 실행합니다: 후보 키워드 수집 → 네이버 뉴스 수집 → 정규화 → 키워드 추출 → 이슈 클러스터링 → 시간대별 집계 → 급상승 판정 → 검색 관심도 검증 → AI 요약 → TOP-10 스냅샷 저장
- 조회 API는 미리 계산된 스냅샷을 읽기만 하는 읽기 전용 REST API입니다 (MVP는 인증 없음)

## API 요약

| 메서드 | 경로 | 설명 |
| --- | --- | --- |
| GET | `/api/v1/trending?category=` | 카테고리별 급상승 이슈 TOP 10 |
| GET | `/api/v1/issues/{id}` | 이슈 상세 (요약, 관련 키워드 포함) |
| GET | `/api/v1/issues/{id}/stats` | 시간대별 언급량 (기본 48시간) |
| GET | `/api/v1/issues/{id}/news` | 관련 뉴스 목록 (페이징) |
| GET | `/health` | 헬스 체크 (마지막 성공 런 경과 포함) |

상세 계약은 [doc/04-API-명세.md](doc/04-API-명세.md) 참고.

## 문서

설계 문서는 `doc/` 디렉토리에 있습니다.

1. [기능 정의서](doc/01-기능-정의서.md) - 기능별 입력/출력, 예외, 상태 변화
2. [시스템 설계서](doc/02-시스템-설계서.md) - 아키텍처, 데이터 흐름, 외부 API 연동
3. [DB 설계](doc/03-DB-설계.md) - ERD, 테이블/컬럼 설명
4. [API 명세](doc/04-API-명세.md) - 엔드포인트, Request/Response, 에러
5. [개발 계획서](doc/05-개발-계획서.md) - 마일스톤, 작업 단위, 우선순위

## 실행

```bash
# 테스트
./gradlew test

# 로컬 실행
./gradlew bootRun
```

### 환경변수

| 변수 | 설명 |
| --- | --- |
| `DATABASE_URL` | PostgreSQL 연결 URL (Railway 주입, `postgres://` → JDBC 변환 필요) |
| `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET` | 네이버 오픈 API 인증 |
| `LLM_API_KEY` | 요약용 LLM API 키 |

## 팀

| 담당 | 이름 |
| --- | --- |
| 백엔드 | 이광석 |
| 앱 | 이민서 |
