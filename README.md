# Korea Pulse Backend

> 지금 대한민국이 무엇에 주목하고 있는지 한눈에

수집한 국내 뉴스에서 관심이 증가하는 이슈를 탐지해 **카테고리·기간별 이슈 순위**와 **이슈 상세(추이·관련 기사·연관 키워드)** 를 제공하는 Korea Pulse의 백엔드 서버입니다. 모든 지표는 **수집한 뉴스 기준**이며, 국민 전체의 관심이나 여론을 측정한 값이 아닙니다.

기준 문서는 [doc/00-프로젝트-정의서-v0.2.md](doc/00-프로젝트-정의서-v0.2.md)이고, 01~05 설계 문서는 이 정의서에 맞춰 정리돼 있습니다.

## 기술 스택

| 구분 | 내용 |
| --- | --- |
| 언어 / 프레임워크 | Java 21, Spring Boot 4.1.1 |
| DB | PostgreSQL (로컬 개발: H2) |
| 캐시 | Redis (랭킹 Sorted Set, 없으면 PostgreSQL 폴백) |
| ORM | Spring Data JPA |
| 배포 | **미확정** (Railway를 기준 후보로 설계) |
| 뉴스 공급원 | **NAVER News Search API** (언급량·증가율·Hot Score의 유일한 입력). `NewsProvider` 인터페이스로 교체 가능 |
| 영상 신호 | **YouTube Data API v3** — 인기 차트 + 뉴스 채널 업로드. `search.list`는 하루 100호출 제한이라 **쓰지 않는다** |
| 검색 관심도 | **Google Trends API** (알파 신청제). 미승인이면 해당 지표만 비활성 |
| 보조 연동 | LLM(카테고리 분류 폴백) |

영상·검색 관심도는 **순위와 Hot Score에 들어가지 않는** 상세 전용 보조 지표입니다. 이슈별 커버리지가 불균등해서입니다.

AI 요약, 회원·알림·결제, B2B 키워드 워치는 MVP 범위 밖입니다.

## 동작 개요

- 스케줄러가 15분(목표 주기)마다 파이프라인을 실행합니다: 후보 키워드 → 뉴스 수집 → 정규화 → 중복 제거 → 키워드 추출 → 이슈 클러스터링 → 카테고리 분류 → 시간 버킷 집계 → Hot Score 계산 → 보조 지표(검색·영상) → 스냅샷 게시
- 조회 API는 게시된 스냅샷만 읽는 읽기 전용 REST API입니다 (MVP는 인증 없음)
- 순위 기간은 1h·6h·24h·7d이며, 현재 구간 `[T-P, T)`와 비교 구간 `[T-2P, T-P)`를 같은 길이로 맞춰 계산합니다

## API 요약

| 메서드 | 경로 | 설명 |
| --- | --- | --- |
| GET | `/api/v1/trends?category=&period=&sort=` | 이슈 순위 (sort: hot·growth·mentions) |
| GET | `/api/v1/topics/{topicId}` | 이슈 상세 (요약 지표·근거·대표 기사) |
| GET | `/api/v1/topics/{topicId}/timeline` | 시간별 추이 (누락 구간과 0을 구분) |
| GET | `/api/v1/topics/{topicId}/articles` | 관련 기사 (중복 그룹 단위 페이징) |
| GET | `/api/v1/topics/{topicId}/related-keywords` | 연관 키워드 |
| GET | `/api/v1/search?q=` | 추적 중인 이슈 검색 |
| GET | `/health` | 헬스 체크 (마지막 성공 런, 케이던스, 수집 지연) |

상세 계약은 [doc/04-API-명세.md](doc/04-API-명세.md) 참고.

## 문서

설계 문서는 `doc/` 디렉토리에 있습니다.

0. [프로젝트 정의서 v0.2](doc/00-프로젝트-정의서-v0.2.md) - 제품 목적, MVP 범위, 지표 정의, 수익 구조 (기준 문서)
1. [기능 정의서](doc/01-기능-정의서.md) - 기능별 입력/출력, 예외, 상태 변화
2. [시스템 설계서](doc/02-시스템-설계서.md) - 아키텍처, 파이프라인 11단계, Redis 랭킹, 쿼터
3. [DB 설계](doc/03-DB-설계.md) - ERD, 테이블/컬럼, 보존 정책
4. [API 명세](doc/04-API-명세.md) - 엔드포인트, Request/Response, 에러
5. [개발 계획서](doc/05-개발-계획서.md) - 준비 단계, 주차별 마일스톤, 검증 게이트, 비용

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
| `DATABASE_URL` | PostgreSQL 연결 URL (`postgres://` → JDBC 변환 필요) |
| `REDIS_URL` | Redis 연결 URL. 없으면 랭킹 캐시를 끄고 PostgreSQL로 조회 |
| `NEWS_PROVIDER` | 사용할 뉴스 공급자 구현 선택 |
| `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET` | 네이버 뉴스 검색 API 인증 |
| `YOUTUBE_API_KEY` | YouTube Data API v3. 없으면 영상 수집·영상 관심도 비활성 |
| `YOUTUBE_NEWS_CHANNEL_IDS` | 업로드를 구독할 뉴스 채널 ID (콤마 구분) |
| `GOOGLE_TRENDS_API_KEY` | Google Trends 알파 자격. 없으면 검색 관심도 비활성 |
| `LLM_API_KEY` / `LLM_BASE_URL` | 카테고리 분류 폴백 |

## 팀

| 담당 | 이름 |
| --- | --- |
| 백엔드 | 이광석 |
| 프론트엔드·디자인 | 이민서 |
