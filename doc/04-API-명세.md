# Korea Pulse 백엔드 API 명세

| 항목 | 내용 |
| --- | --- |
| 문서 번호 | 04 |
| 대상 | Korea Pulse MVP 백엔드 (Java 21, Spring Boot 3.5.x, PostgreSQL, Railway) |
| 근거 문서 | PRD "사이드 프로젝트.txt" §8, §13. 01-기능-정의서(출력 항목), 03-DB-설계(컬럼) |
| 관련 문서 | 01-기능-정의서, 02-시스템-설계서, 03-DB-설계, 05-개발-계획서 |
| 작성 기준일 | 2026-09-13 |

이 문서는 앱이 호출하는 HTTP 엔드포인트의 형식을 확정한다. 응답 필드명은 01 문서의 출력 항목을 그대로 쓰고, 각 필드가 03 문서의 어느 컬럼에서 오는지 표로 밝힌다. "무엇을 왜 제공하는가"는 01이, "어디에 저장하는가"는 03이 다루므로 여기서는 반복하지 않는다.

---

## 1. 공통 규약

### 1.1 기본 경로와 버전

| 항목 | 값 |
| --- | --- |
| Base path | `/api/v1` |
| 프로토콜 | HTTPS only (Railway 도메인이 TLS 종단) |
| 메서드 | GET만 제공한다. MVP 백엔드는 읽기 전용이며 쓰기 엔드포인트가 없다 |
| 인증 | 없음. 모든 요청은 익명이다 (01 문서 1.2절). 인증이 필요해지는 시점은 회원 기능이 로드맵에 들어올 때이고, 그때 `/api/v2`로 분리한다 |
| 헬스 체크 | `/health`는 base path 밖에 둔다. 운영 도구가 호출하는 경로라 버전을 태우지 않는다 |

내부 디버그용 엔드포인트(런 상태 조회 등)는 `/internal/**` 아래에 두고 Railway 프라이빗 네트워크에서만 접근하게 한다. 이 문서의 범위 밖이며 앱은 호출하지 않는다.

### 1.2 요청 형식

- 쿼리 파라미터만 쓴다. GET 요청에 본문은 없다.
- 파라미터 이름은 camelCase, 값은 대소문자를 구분한다. `category=economy`는 400이다.
- `Accept` 헤더는 무시하고 항상 `application/json; charset=utf-8`로 응답한다.

### 1.3 응답 형식

- 본문은 JSON 객체 하나다. 배열을 최상위로 두지 않는다. 목록은 항상 `items` 필드 안에 넣어 메타 정보(개수, 페이지, 계산 시각)를 함께 실을 자리를 남긴다.
- 필드명은 camelCase. 03 문서의 snake_case 컬럼을 응답 직전에 변환한다 (`surge_rate` → `surgeRate`).
- 값이 없는 필드는 생략하지 않고 `null`로 내보낸다. 클라이언트가 "필드가 없다"와 "값이 없다"를 구분할 필요가 없도록 한다.
- 정수는 JSON number, 소수(`previousAvgCount`, `surgeRate`)는 소수점 둘째 자리까지 반올림한 number다. `surgeRate`는 반올림된 previousAvgCount로 계산해 반올림하며, 스냅샷 저장값을 추가 변환 없이 반환한다. `surgeRate` 1.82는 +182%를 뜻한다.

### 1.4 시각 표현

모든 시각은 KST ISO 8601 문자열이고 오프셋 `+09:00`을 항상 붙인다. 저장은 UTC지만(03 문서), 응답 직전에 `Asia/Seoul`로 변환한다.

| 종류 | 형식 | 예 |
| --- | --- | --- |
| 일시(datetime) | `yyyy-MM-dd'T'HH:mm:ss+09:00` | `2026-09-13T14:31:40+09:00` |
| 시간 버킷(hour) | 위와 같되 분·초는 항상 00 | `2026-09-13T13:00:00+09:00` |
| 일자(date) | `yyyy-MM-dd` | `2026-09-12` |

시간 버킷은 UTC 정각 기준이라 KST로 바꿔도 정각이다. `searchInterestAsOf`처럼 날짜만 의미 있는 값은 오프셋 없이 `yyyy-MM-dd`로 준다.

### 1.5 캐시

파이프라인은 30분마다 돌고 그 사이 스냅샷은 바뀌지 않는다. 그래서 스냅샷을 읽는 응답에는 다음 런 예상 시각까지 남은 초를 `Cache-Control: public, max-age=<초>`로 실어 앱과 중간 캐시의 재요청을 줄인다.

| 엔드포인트 | Cache-Control | 근거 |
| --- | --- | --- |
| `/api/v1/trending` | `public, max-age=<nextExpectedAt - now, 초>` (최소 30, 최대 1800) | 스냅샷은 다음 SUCCESS 런까지 불변 |
| `/api/v1/issues/{id}` | `public, max-age=60` | 조회 시점 계산값(mentionCount, surgeRate)이 있어 짧게 둔다 |
| `/api/v1/issues/{id}/stats` | `public, max-age=300` | 버킷 단위 데이터라 5분 안에 크게 바뀌지 않는다 |
| `/api/v1/issues/{id}/news` | `public, max-age=300` | 런당 한 번 늘어난다 |
| `/health` | `no-store` | 운영 신호는 항상 최신이어야 한다 |
| 4xx, 5xx 전체 | `no-store` | 오류를 캐시하지 않는다 |

마지막 런이 FAILED여서 직전 SUCCESS 스냅샷을 서빙 중일 때 `nextExpectedAt`은 이미 과거일 수 있다. 이때 `max-age`는 하한 30으로 고정해 앱이 30초마다 새 스냅샷을 확인하게 한다.

### 1.6 공통 에러 형식

모든 4xx, 5xx 응답은 같은 모양이다.

```json
{
  "code": "INVALID_CATEGORY",
  "message": "지원하지 않는 category 값입니다: economy",
  "timestamp": "2026-09-13T14:35:02+09:00"
}
```

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| code | string | 기계가 분기할 상수. 7절의 목록 중 하나 |
| message | string | 사람이 읽는 한국어 설명. 앱이 그대로 표시해도 되지만 문구는 예고 없이 바뀔 수 있으므로 분기에는 `code`를 쓴다 |
| timestamp | datetime | 오류가 발생한 서버 시각 (KST) |

`message`에 스택 트레이스, SQL, 내부 클래스명을 넣지 않는다. 500의 `message`는 고정 문구다.

### 1.7 엔드포인트 요약

| 메서드 | 경로 | 목적 | 01 문서 대응 |
| --- | --- | --- | --- |
| GET | `/api/v1/trending?category=` | 카테고리별 급상승 TOP-10 | 기능 1 |
| GET | `/api/v1/issues/{id}` | 이슈 상세 + 관련 키워드 + AI 요약 | 기능 2, 3.2.1 |
| GET | `/api/v1/issues/{id}/stats?from=&to=` | 시간대별 언급량 (기본 48h) | 기능 2, 3.2.2 |
| GET | `/api/v1/issues/{id}/news?page=&size=` | 관련 뉴스 목록 (페이징, 링크아웃) | 기능 2, 3.2.3 |
| GET | `/health` | 마지막 SUCCESS 런 경과 분 | 02 문서 배포 구성 |

PRD §13의 사용자 흐름은 `trending` → `issues/{id}` → (`stats`, `news` 병렬) → 외부 링크 이동 순서로 이 네 엔드포인트를 차례로 탄다.

---

## 2. GET /api/v1/trending

가장 최근 SUCCESS 런의 스냅샷에서 카테고리 하나의 TOP-10을 돌려준다. 03 문서 `trending_snapshot_entry`를 `(run_id, category)`로 읽어 `rank` 오름차순으로 최대 10행을 싣는다. 불변·격리 보장은 스냅샷에 저장된 순위·지표에 한정되며, 이슈명은 라이브 issue.name을 조인한다.

### 2.1 요청

| 파라미터 | 위치 | 필수 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- | --- | --- |
| category | query | 아니오 | string | `ALL` | `ALL`, `ECONOMY`, `SPORTS`, `ENTERTAINMENT`, `POLITICS_SOCIETY`, `IT_SCIENCE`, `ETC` 중 하나 |

```http
GET /api/v1/trending?category=ECONOMY HTTP/1.1
Host: api.koreapulse.example
Accept: application/json
```

### 2.2 응답 200

```json
{
  "category": "ECONOMY",
  "computedAt": "2026-09-13T14:31:40+09:00",
  "nextExpectedAt": "2026-09-13T15:01:40+09:00",
  "items": [
    {
      "rank": 1,
      "issueId": 48377,
      "issueName": "코스피 3400 돌파",
      "mentionCount": 12,
      "previousAvgCount": 0.0,
      "surgeRate": 12.0,
      "isNew": true,
      "searchInterestIndex": null,
      "searchInterestAsOf": null,
      "sparkline": [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 12]
    },
    {
      "rank": 2,
      "issueId": 48213,
      "issueName": "기준금리 동결 한국은행",
      "mentionCount": 41,
      "previousAvgCount": 14.5,
      "surgeRate": 1.83,
      "isNew": false,
      "searchInterestIndex": 87,
      "searchInterestAsOf": "2026-09-12",
      "sparkline": [3, 5, 4, 6, 9, 11, 12, 10, 8, 14, 13, 15, 12, 16, 18, 17, 19, 21, 20, 18, 22, 25, 29, 41]
    }
  ]
}
```

예시는 현재 케이던스 30분 기준이다. sparkline은 현재 버킷과 그 앞 23개만 담으므로 previousAvgCount 계산에는 배열 직전 버킷 1개가 더 필요하다. 예시에서 그 값은 issueId 48377이 0, 48213이 21이다. 따라서 각각 (0 + 0) / 24 = 0.0, (21 + 327) / 24 = 14.5이며, 증가율은 12.0과 1.83이다. isNew를 포함한 모든 항목은 surgeRate 내림차순, 동률 시 mentionCount 내림차순으로 순위를 정한다.

응답 헤더:

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Cache-Control: public, max-age=1620
```

### 2.3 응답 필드

| 필드 | 타입 | nullable | 03 컬럼 | 설명 |
| --- | --- | --- | --- | --- |
| category | string | 아니오 | trending_snapshot_entry.category | 요청한 코드 그대로 |
| computedAt | datetime | 아니오 | pipeline_run.finished_at | 스냅샷을 만든 런의 완료 시각 |
| nextExpectedAt | datetime | 아니오 | pipeline_run.finished_at + 현재 케이던스(30분, 축퇴 시 60분) | 다음 갱신 예상 시각. Cache-Control 계산 기준 |
| items | array | 아니오 | | 0~10개, rank 오름차순. 순위 대상이 없으면 빈 배열 |
| items[].rank | int | 아니오 | trending_snapshot_entry.rank | 1~10 |
| items[].issueId | long | 아니오 | trending_snapshot_entry.issue_id | `/issues/{id}` 호출에 쓴다 |
| items[].issueName | string | 아니오 | issue.name | 대표 키워드 최대 3개를 공백으로 이은 문자열 |
| items[].mentionCount | int | 아니오 | trending_snapshot_entry.mention_count | 직전 완료 버킷 1시간의 기사 수 |
| items[].previousAvgCount | float | 아니오 | trending_snapshot_entry.previous_avg_count | 직전 24개 버킷 시간당 평균. 소수 둘째 자리 |
| items[].surgeRate | float | 아니오 | trending_snapshot_entry.surge_rate | (mentionCount - previousAvgCount) / max(previousAvgCount, 1.0). 1.82 = +182% |
| items[].isNew | boolean | 아니오 | trending_snapshot_entry.is_new | 베이스라인 부족 시 true (NEW 배지) |
| items[].searchInterestIndex | int (0~100) | 예 | trending_snapshot_entry.search_interest_index | 최근 가용 일자 ratio(NUMERIC(5,2))를 ROUND()한 정수. 미조회·실패면 null |
| items[].searchInterestAsOf | date | 예 | trending_snapshot_entry.search_interest_as_of | 그 값이 어느 날짜 것인지. 보통 전일. `searchInterestIndex`와 항상 함께 있거나 함께 null |
| items[].sparkline | int[24] | 아니오 | trending_snapshot_entry.sparkline | 오래된 버킷이 앞, 마지막 원소가 현재 버킷. 결측은 0 |

앱은 `searchInterestIndex`를 표시할 때 `searchInterestAsOf`를 반드시 함께 보여야 한다(01 문서 6.2절). 첫 번째 예시처럼 `isNew=true`인 항목도 `surgeRate`를 최소 분모 규칙(1.0)으로 계산하며 동일한 순위 기준을 적용한다.

### 2.4 오류

| 상태 | code | 상황 |
| --- | --- | --- |
| 400 | `INVALID_CATEGORY` | 7종 코드에 없는 값. 소문자, 빈 문자열 포함 |
| 503 | `NO_SNAPSHOT_AVAILABLE` | SUCCESS 런이 한 번도 없음 (최초 배포 직후 또는 연속 실패로 스냅샷 부재). `Retry-After: 300` 헤더 동봉 |

마지막 런이 FAILED인 경우는 오류가 아니다. 직전 SUCCESS 스냅샷을 200으로 돌려주며, 앱은 `computedAt`이 30분 이상 과거인지로 오래됨을 판단한다.

---

## 3. GET /api/v1/issues/{id}

이슈 하나의 기본 정보, 조회 시점 언급량과 증가율, 관련 키워드 top10, AI 요약을 돌려준다. `trending`의 `mentionCount`, `surgeRate`는 런 시점에 고정된 값이고 여기 값은 조회 시점에 `issue_hourly_stat`으로 계산한다. 런 경계를 넘으면 상세가 먼저 최신 값을 보일 수 있고, 이 차이는 허용한다(01 문서 3.2.1절).

### 3.1 요청

| 파라미터 | 위치 | 필수 | 타입 | 설명 |
| --- | --- | --- | --- | --- |
| id | path | 예 | long | `trending`이 돌려준 `issueId` |

```http
GET /api/v1/issues/48213 HTTP/1.1
Host: api.koreapulse.example
Accept: application/json
```

### 3.2 응답 200

```json
{
  "issueId": 48213,
  "issueName": "기준금리 동결 한국은행",
  "category": "ECONOMY",
  "status": "ACTIVE",
  "mentionCount": 43,
  "surgeRate": 1.97,
  "searchInterestIndex": 87,
  "searchInterestAsOf": "2026-09-12",
  "keywords": ["기준금리", "동결", "한국은행", "금통위", "물가", "환율", "가계부채", "총재", "연준", "인하"],
  "summary": "한국은행 금융통화위원회가 기준금리를 현 수준에서 동결했다. 물가 둔화 속도와 가계부채 증가세를 함께 고려한 결정으로 알려졌다. 시장은 연내 인하 가능성을 두고 총재 발언에 주목하고 있다.",
  "summaryGeneratedAt": "2026-09-13T14:31:22+09:00",
  "firstSeenAt": "2026-09-13T09:31:05+09:00",
  "lastMentionedAt": "2026-09-13T14:27:00+09:00"
}
```

요약이 아직 없는 이슈는 `summary`와 `summaryGeneratedAt`이 함께 null이다.

```json
{
  "issueId": 48377,
  "issueName": "코스피 3400 돌파",
  "category": "ECONOMY",
  "status": "ACTIVE",
  "mentionCount": 12,
  "surgeRate": 12.0,
  "searchInterestIndex": null,
  "searchInterestAsOf": null,
  "keywords": ["코스피", "3400", "돌파", "외국인", "반도체", "사상", "최고치"],
  "summary": null,
  "summaryGeneratedAt": null,
  "firstSeenAt": "2026-09-13T13:31:08+09:00",
  "lastMentionedAt": "2026-09-13T14:29:00+09:00"
}
```

### 3.3 응답 필드

| 필드 | 타입 | nullable | 03 컬럼 | 설명 |
| --- | --- | --- | --- | --- |
| issueId | long | 아니오 | issue.id | |
| issueName | string | 아니오 | issue.name | `trending`과 동일 |
| category | string | 아니오 | issue.category | `ALL`을 제외한 6종 중 하나 |
| status | string | 아니오 | issue.status | `ACTIVE` 또는 `DORMANT` |
| mentionCount | int | 아니오 | issue_hourly_stat.mention_count (직전 완료 버킷) | 조회 시점 계산 |
| surgeRate | float | 아니오 | issue_hourly_stat 최근 25개 버킷으로 계산 | `trending`과 같은 규칙 |
| searchInterestIndex | int (0~100) | 예 | search_interest_daily.ratio (issue_keyword의 article_count 최다 대표 키워드(primary) 기준 최신 가용 일자) | NUMERIC(5,2) → int: ROUND(ratio). 예: 87.65 → 88 |
| searchInterestAsOf | date | 예 | search_interest_daily.date | `searchInterestIndex`와 함께 있거나 함께 null |
| keywords | string[] | 아니오 | issue_keyword.keyword (article_count 내림차순 상위 10) | 이슈명에 쓰인 키워드 포함. 최대 10개 |
| summary | string | 예 | issue.summary | 2~3문장 한국어. 미생성이면 null |
| summaryGeneratedAt | datetime | 예 | issue.summary_generated_at | `summary`가 null이면 null |
| firstSeenAt | datetime | 아니오 | issue.first_seen_at | PRD §8 "이슈 발생 시점" |
| lastMentionedAt | datetime | 아니오 | issue.last_mentioned_at | 소속 기사 중 가장 최신 발행 시각 |

### 3.4 오류

| 상태 | code | 상황 |
| --- | --- | --- |
| 400 | `INVALID_PARAMETER` | `id`가 양의 정수가 아님 (`/issues/abc`, `/issues/-1`) |
| 404 | `ISSUE_NOT_FOUND` | 해당 id의 이슈가 없음 |

`DORMANT` 이슈는 404가 아니다. 스냅샷 이력에서 도달할 수 있으므로 200으로 주고 `status`로 구분한다.

---

## 4. GET /api/v1/issues/{id}/stats

이슈의 시간대별 언급량을 1시간 버킷 시계열로 돌려준다. 03 문서 `issue_hourly_stat`을 `(issue_id, bucket_hour)` 범위로 읽고, 데이터가 없는 버킷은 0으로 채워 구간 안의 모든 버킷이 빠짐없이 나오게 한다.

### 4.1 요청

| 파라미터 | 위치 | 필수 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- | --- | --- |
| id | path | 예 | long | | 이슈 식별자 |
| from | query | 아니오 | datetime (ISO 8601) | 현재 시각을 정각으로 내린 값 - 48시간 | 구간 시작 (포함). 분·초가 있으면 정각으로 내린다 |
| to | query | 아니오 | datetime (ISO 8601) | 현재 시각을 정각으로 내린 값 | 구간 끝 (제외). 분·초가 있으면 정각으로 내린다 |

`from`, `to`는 오프셋을 포함한 ISO 8601이면 어떤 시간대로 보내도 된다. 오프셋을 생략하면 KST로 해석한다. URL에서는 `+`를 `%2B`로 인코딩해야 한다.

조회 구간은 반개구간 `[from, to)`로 from 포함, to 제외다. 기본값 구간(48시간)이면 `points`는 48개다. `from`과 `to`의 폭은 최대 7일(168버킷)이다.

```http
GET /api/v1/issues/48213/stats HTTP/1.1
Host: api.koreapulse.example
Accept: application/json
```

구간을 직접 지정하는 경우:

```http
GET /api/v1/issues/48213/stats?from=2026-09-13T09:00:00%2B09:00&to=2026-09-13T14:00:00%2B09:00 HTTP/1.1
Host: api.koreapulse.example
Accept: application/json
```

### 4.2 응답 200

아래는 위 두 번째 요청(09:00~14:00, 반개구간 5시간·5버킷)의 응답이다. 기본 요청이면 같은 모양으로 `points`가 48개다.

```json
{
  "issueId": 48213,
  "from": "2026-09-13T09:00:00+09:00",
  "to": "2026-09-13T14:00:00+09:00",
  "points": [
    { "hour": "2026-09-13T09:00:00+09:00", "mentionCount": 18 },
    { "hour": "2026-09-13T10:00:00+09:00", "mentionCount": 22 },
    { "hour": "2026-09-13T11:00:00+09:00", "mentionCount": 25 },
    { "hour": "2026-09-13T12:00:00+09:00", "mentionCount": 29 },
    { "hour": "2026-09-13T13:00:00+09:00", "mentionCount": 41 }
  ]
}
```

### 4.3 응답 필드

| 필드 | 타입 | nullable | 03 컬럼 | 설명 |
| --- | --- | --- | --- | --- |
| issueId | long | 아니오 | issue.id | |
| from | datetime | 아니오 | | 실제 적용된 구간 시작 (포함, 정각으로 내린 값) |
| to | datetime | 아니오 | | 실제 적용된 구간 끝 (제외) |
| points | array | 아니오 | | `from <= hour < to`인 버킷을 1시간 간격, 오름차순으로 반환. 빈 버킷도 0으로 채워 넣는다 |
| points[].hour | datetime | 아니오 | issue_hourly_stat.bucket_hour | 버킷 시작 시각. 분·초는 항상 00 |
| points[].mentionCount | int | 아니오 | issue_hourly_stat.mention_count | 해당 버킷에 발행된 소속 기사 수 |

명시한 to가 현재 정각보다 뒤라면 마지막 원소는 진행 중인 버킷일 수 있어 아직 채워지지 않은 값이다. 기본 요청과 위 예시(to=14:00)는 14시 진행 중 버킷을 제외하고 13시 버킷까지 반환한다. 순위 계산은 이와 별도로 직전 완성 버킷만 사용한다. `issue_hourly_stat` 보존 기간(30일)을 넘는 구간은 0으로 채워진다.

### 4.4 오류

| 상태 | code | 상황 |
| --- | --- | --- |
| 400 | `INVALID_PARAMETER` | `id`가 양의 정수가 아님. `from` 또는 `to`가 ISO 8601이 아님. `from > to`. 폭이 7일 초과 |
| 404 | `ISSUE_NOT_FOUND` | 해당 id의 이슈가 없음 |

---

## 5. GET /api/v1/issues/{id}/news

이슈에 연결된 기사를 발행 시각 내림차순으로 페이징해 돌려준다. 03 문서 `issue_article`을 issue_id로 필터링하고 `article` 조인 후 `article.published_at DESC`로 정렬한다.

이 응답에는 기사 본문이 없고 `description`도 없다. 제목과 두 개의 링크만 주고, 읽기는 네이버 뉴스 또는 언론사 원문 페이지에서 이루어진다(01 문서 8절 법적 제약). 출처 표기는 서버가 `source` 고정 필드로 실어 앱이 빠뜨리지 못하게 한다.

### 5.1 요청

| 파라미터 | 위치 | 필수 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- | --- | --- |
| id | path | 예 | long | | 이슈 식별자 |
| page | query | 아니오 | int | 0 | 0부터 시작하는 페이지 번호 |
| size | query | 아니오 | int | 10 | 페이지 크기. 1~50 |

```http
GET /api/v1/issues/48213/news?page=0&size=10 HTTP/1.1
Host: api.koreapulse.example
Accept: application/json
```

### 5.2 응답 200

```json
{
  "issueId": 48213,
  "totalCount": 137,
  "page": 0,
  "size": 10,
  "source": "출처: 네이버 뉴스",
  "items": [
    {
      "title": "한은, 기준금리 연 2.50%로 동결... 가계부채 증가세 경계",
      "originallink": "https://www.example-news.co.kr/economy/2026/09/13/0001234567",
      "naverLink": "https://n.news.naver.com/mnews/article/001/0015123456",
      "publisher": "예시일보",
      "publishedAt": "2026-09-13T14:27:00+09:00"
    },
    {
      "title": "금통위 \"물가 안정 흐름 지속... 인하 시점은 신중히 판단\"",
      "originallink": "https://biz.example-daily.com/article/20260913140812",
      "naverLink": "https://n.news.naver.com/mnews/article/002/0002456789",
      "publisher": "예시경제",
      "publishedAt": "2026-09-13T14:08:12+09:00"
    },
    {
      "title": "[속보] 한국은행 기준금리 동결 결정",
      "originallink": "https://news.example-broadcast.kr/view/20260913135505",
      "naverLink": "https://news.example-broadcast.kr/view/20260913135505",
      "publisher": null,
      "publishedAt": "2026-09-13T13:55:05+09:00"
    }
  ]
}
```

예시에서는 지면상 3건만 보였다. 실제 `size=10` 응답은 `items`가 10개다.

### 5.3 응답 필드

| 필드 | 타입 | nullable | 03 컬럼 | 설명 |
| --- | --- | --- | --- | --- |
| issueId | long | 아니오 | issue.id | |
| totalCount | int | 아니오 | count(issue_article where issue_id) | 이슈에 연결된 기사 전체 수. 보존 기간(14일) 만료로 삭제된 기사는 빠진다 |
| page | int | 아니오 | | 적용된 페이지 번호 |
| size | int | 아니오 | | 적용된 페이지 크기 |
| source | string | 아니오 | (고정값) | 항상 `"출처: 네이버 뉴스"`. 앱은 목록 근처에 표기한다 |
| items | array | 아니오 | | 발행 시각 내림차순. 0~size개 |
| items[].title | string | 아니오 | article.title | HTML 태그를 제거한 제목 |
| items[].originallink | string | 아니오 | article.originallink | 언론사 원문 URL. 기본 링크아웃 대상 |
| items[].naverLink | string | 아니오 | article.naver_link (NULL이면 originallink로 대체) | 네이버 뉴스 URL. 원문과 같으면 `originallink`와 동일 값 |
| items[].publisher | string | 예 | article.publisher | 언론사 표시명. 도메인 매핑이 없으면 null |
| items[].publishedAt | datetime | 아니오 | article.published_at | KST 변환 값 |

`naverLink`는 항상 값이 있다. 03 문서에서 `article.naver_link`가 NULL인 경우(원문과 동일)에는 응답 직전에 `originallink`로 채운다. 두 링크 값은 동일할 수 있다.

### 5.4 오류

| 상태 | code | 상황 |
| --- | --- | --- |
| 400 | `INVALID_PARAMETER` | `id`가 양의 정수가 아님. `page < 0`. `size < 1` 또는 `size > 50`. 정수가 아닌 값 |
| 404 | `ISSUE_NOT_FOUND` | 해당 id의 이슈가 없음 |

범위를 넘는 `page`(예: totalCount 137에 page=50)는 오류가 아니다. `items`가 빈 배열이고 `totalCount`는 정상 값이다. 기사가 전부 보존 기간을 넘겨 삭제된 이슈도 `totalCount: 0`, `items: []`로 200이다.

---

## 6. GET /health

Railway 헬스 체크와 운영자가 보는 단일 신호다. Spring Boot Actuator의 health 엔드포인트를 `/health` 경로로 노출하고, 02 문서 배포 구성에 따라 커스텀 `pipeline` 인디케이터가 마지막 SUCCESS 런 경과 분을 싣는다.

### 6.1 요청

```http
GET /health HTTP/1.1
Host: api.koreapulse.example
Accept: application/json
```

### 6.2 응답 200

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP"
    },
    "pipeline": {
      "status": "UP",
      "details": {
        "lastSuccessAt": "2026-09-13T14:31:40+09:00",
        "minutesSinceLastSuccess": 4,
        "lastRunStatus": "SUCCESS",
        "cadenceMinutes": 30
      }
    }
  }
}
```

SUCCESS 런이 한 번도 없을 때:

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP"
    },
    "pipeline": {
      "status": "UP",
      "details": {
        "lastSuccessAt": null,
        "minutesSinceLastSuccess": null,
        "lastRunStatus": "FAILED",
        "cadenceMinutes": 30
      }
    }
  }
}
```

### 6.3 응답 필드

| 필드 | 타입 | nullable | 03 컬럼 | 설명 |
| --- | --- | --- | --- | --- |
| status | string | 아니오 | | 전체 상태. DB 연결이 되는 한 `UP`. 파이프라인 지연은 재시작으로 고쳐지지 않으므로 `DOWN`으로 내려 Railway가 컨테이너를 재시작하게 만들지 않는다 |
| components.db.status | string | 아니오 | | DataSource 연결 확인 결과. 실패 시 `DOWN`이고 전체 `status`도 `DOWN` |
| components.pipeline.status | string | 아니오 | | 항상 `UP` |
| components.pipeline.details.lastSuccessAt | datetime | 예 | pipeline_run.finished_at (status = SUCCESS 최신 1건) | 마지막 SUCCESS 런 완료 시각 |
| components.pipeline.details.minutesSinceLastSuccess | int | 예 | now - 위 값 | 마지막 SUCCESS 런 이후 경과 분. SUCCESS 런이 없으면 null. 90을 넘으면 연속 3회 이상 실패 중이라는 뜻이다 |
| components.pipeline.details.lastRunStatus | string | 예 | pipeline_run.status (최신 1건) | `RUNNING`, `SUCCESS`, `FAILED`. 런이 한 번도 없으면 null |
| components.pipeline.details.cadenceMinutes | int | 아니오 | (설정값) | 런 간격. `minutesSinceLastSuccess`를 해석하는 기준 |

### 6.4 오류

DB 연결이 끊긴 경우에만 503이며, 이때는 Actuator 형식(`status: "DOWN"`)으로 응답한다. 7절의 공통 에러 형식은 쓰지 않는다. 이 엔드포인트는 앱이 호출하지 않으므로 앱 쪽 분기가 필요 없다.

---

## 7. 주요 에러

### 7.1 에러 코드 표

| 상태 | code | 발생 엔드포인트 | 상황 | 응답 계약 |
| --- | --- | --- | --- | --- |
| 400 | `INVALID_CATEGORY` | trending | 7종 코드 밖의 category 값 | 허용 category는 2.1절 코드 7종 |
| 400 | `INVALID_PARAMETER` | issues/{id}, stats, news | id가 양의 정수가 아님. from/to 형식 오류, from > to, 폭 7일 초과. page 음수, size 1~50 밖 | 파라미터 검증 실패, 공통 에러 형식 반환 |
| 404 | `ISSUE_NOT_FOUND` | issues/{id}, stats, news | 존재하지 않는 이슈 id | 해당 id의 이슈가 없음을 반환. DORMANT는 해당하지 않음 |
| 429 | `RATE_LIMITED` | 전체 (`/health` 제외) | 같은 IP의 요청이 분당 한도(기본 60회)를 초과 | `Retry-After` 헤더(초 단위)와 공통 에러 형식 반환. 사용량 과금 방어 목적(05 문서 M3-09) |
| 500 | `INTERNAL_ERROR` | 전체 | 처리되지 않은 예외, DB 오류 | 내부 세부 없이 공통 에러 형식 반환 |
| 503 | `NO_SNAPSHOT_AVAILABLE` | trending | SUCCESS 런이 한 번도 없어 서빙할 스냅샷이 없음 | `Retry-After: 300` 헤더와 공통 에러 형식 반환 |

`issues/{id}`, `stats`, `news`는 SUCCESS 런이 없어도 503을 내지 않는다. 이슈는 런 SUCCESS와 무관하게 존재할 수 있으므로 있으면 200, 없으면 404다(01 문서 9.5절).

### 7.2 예시 본문

400, 잘못된 카테고리:

```json
{
  "code": "INVALID_CATEGORY",
  "message": "지원하지 않는 category 값입니다: economy. 허용값: ALL, ECONOMY, SPORTS, ENTERTAINMENT, POLITICS_SOCIETY, IT_SCIENCE, ETC",
  "timestamp": "2026-09-13T14:35:02+09:00"
}
```

429, 요청 한도 초과:

```json
{
  "code": "RATE_LIMITED",
  "message": "요청이 너무 잦습니다. 잠시 후 다시 시도하세요",
  "timestamp": "2026-09-13T14:35:40+09:00"
}
```

400, 통계 구간 초과:

```json
{
  "code": "INVALID_PARAMETER",
  "message": "from과 to의 간격은 최대 7일입니다",
  "timestamp": "2026-09-13T14:36:11+09:00"
}
```

404, 없는 이슈:

```json
{
  "code": "ISSUE_NOT_FOUND",
  "message": "이슈를 찾을 수 없습니다: 99999999",
  "timestamp": "2026-09-13T14:37:45+09:00"
}
```

500, 서버 오류:

```json
{
  "code": "INTERNAL_ERROR",
  "message": "일시적인 서버 오류입니다. 잠시 후 다시 시도해 주세요",
  "timestamp": "2026-09-13T14:38:20+09:00"
}
```

503, 스냅샷 없음 (`Retry-After: 300` 헤더 동봉):

```json
{
  "code": "NO_SNAPSHOT_AVAILABLE",
  "message": "아직 집계된 데이터가 없습니다. 잠시 후 다시 시도해 주세요",
  "timestamp": "2026-09-13T14:39:03+09:00"
}
```

### 7.3 오류 응답 헤더

| 상태 | 헤더 |
| --- | --- |
| 모든 4xx, 5xx | `Cache-Control: no-store`, `Content-Type: application/json; charset=utf-8` |
| 503 | 위 항목 + `Retry-After: 300` |

---

## 8. 다른 문서와의 대응

| 이 문서 | 대응 |
| --- | --- |
| 2절 `trending` 응답 items[] 필드 | 01 문서 2.2절 출력 표, 03 문서 3.8 `trending_snapshot_entry` |
| 3절 `issues/{id}` 응답 | 01 문서 3.2.1절, 03 문서 3.4 `issue`, 3.5 `issue_keyword`, 3.9 `search_interest_daily` |
| 4절 `stats` 응답 points[] | 01 문서 3.2.2절, 03 문서 3.7 `issue_hourly_stat` |
| 5절 `news` 응답 items[] | 01 문서 3.2.3절과 8절(법적 제약), 03 문서 3.2 `article`, 3.6 `issue_article` |
| 6절 `health` details | 02 문서 배포 구성(HealthIndicator), 03 문서 3.1 `pipeline_run` |
| 1.5절 Cache-Control | 02 문서 스케일 가정(다음 런 예상 시각까지 max-age) |
| 7절 400/404/503 조건 | 01 문서 2.4절, 3.3절, 9.5절 예외 표 |
| 시각 규약 (+09:00) | 01 문서 1.4절, 02 문서 타임존, 03 문서 설계 원칙(UTC 저장) |
