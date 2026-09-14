# Korea Pulse 백엔드 API 명세

| 항목 | 내용 |
| --- | --- |
| 문서 번호 | 04 |
| 문서 버전 | v0.2 정합 (2026-09-14) |
| 대상 | Korea Pulse MVP 백엔드 (Java 21, Spring Boot 4.1.1, PostgreSQL, Redis) |
| 근거 문서 | 00-프로젝트-정의서-v0.2 §10(API 계약 초안), §7(화면), §8(지표). 01-기능-정의서, 03-DB-설계 |
| 관련 문서 | 00-프로젝트-정의서-v0.2, 01-기능-정의서, 02-시스템-설계서, 03-DB-설계, 05-개발-계획서 |

이 문서는 웹 프론트엔드가 호출하는 HTTP 엔드포인트 형식을 확정한다. 필드명은 01 문서의 출력 항목을, 출처 컬럼은 03 문서를 따른다. API 변경은 이 문서와 예시 응답을 함께 갱신한 뒤에만 한다(정의서 §4).

---

## 1. 공통 규약

### 1.1 기본 경로와 버전

| 항목 | 값 |
| --- | --- |
| Base path | `/api/v1` |
| 프로토콜 | HTTPS |
| 메서드 | GET만 제공한다. MVP 백엔드는 읽기 전용이다 |
| 인증 | 없음. 모든 요청은 익명이다. 회원 기능이 생기면 `/api/v2`로 분리한다 |
| 헬스 체크 | `/health`는 base path 밖에 둔다 |

내부 디버그 엔드포인트는 `/internal/**`에 두고 외부에 노출하지 않는다. 프론트는 호출하지 않는다.

### 1.2 요청 형식

- 쿼리 파라미터만 쓴다. GET 요청에 본문은 없다.
- 파라미터 이름은 camelCase. `category`·`sort`는 대소문자를 구분하며 `category=economy`는 400이다. `period` 값(`1h`, `24h`, `7d`)은 소문자 고정이다.
- 항상 `application/json; charset=utf-8`로 응답한다.

### 1.3 응답 형식

- 본문은 JSON 객체 하나다. 목록은 항상 `items` 안에 넣는다.
- 필드명은 camelCase다.
- 값이 없는 필드는 생략하지 않고 `null`로 내보낸다.
- 소수는 소수점 둘째 자리까지 반올림한 number다. `growthRate`는 퍼센트 값이며 `182.35`는 +182.35%를 뜻한다. `hotScore`는 0~100이다.
- 모든 조회 응답은 `meta` 객체를 포함한다 (1.4절).

### 1.4 공통 메타 (meta)

정의서 §10의 공통 메타를 그대로 구현한다.

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| snapshotId | long nullable | 응답이 읽은 스냅샷. 스냅샷과 무관한 응답(`/search`, `/topics/{id}/articles`)은 null |
| generatedAt | datetime nullable | 스냅샷 계산 완료 시각 |
| dataThrough | datetime nullable | 집계 기준 시점 T |
| period | string nullable | 적용된 기간 |
| timezone | string | 항상 `Asia/Seoul` |
| dataStatus | string | `fresh` / `stale` / `insufficient` (01 문서 2.5절) |
| scoreVersion | string nullable | Hot Score 공식 버전 |
| availablePeriods | string[] | 비교 구간까지 확보된 기간. 프론트는 이 목록만 활성화한다 |
| coveredCategories | string[] | 실제 수집·분류가 이루어지는 카테고리 |
| notices | string[] | 폴백·저하 상태 안내 코드. 1.9절 목록 |

순위와 상세를 같은 `snapshotId`로 읽으면 화면 전체가 같은 기준을 쓴다. 상세 요청에 `snapshotId`를 넘기면 그 스냅샷을 고정한다.

### 1.9 notices 코드

| 코드 | 의미 | 발생 위치 |
| --- | --- | --- |
| `SNAPSHOT_FALLBACK` | 요청한 `snapshotId`가 없거나 만료돼 최신 게시 스냅샷으로 폴백 | topics/* |
| `SORT_FALLBACK_MENTIONS` | `sort=growth` 요청을 mentions 순서로 폴백 | trends |
| `PROVIDER_PARTIAL` | 외부 공급자 보강 호출 실패로 저장분만으로 응답 | topics/{id}/articles |
| `COVERAGE_DEGRADED` | 현재 구간의 결손 비율이 임계치(10%)를 넘음. 지표는 제공하되 과소 집계될 수 있다 | trends, topics/{id}, timeline |
| `SCORE_UNAVAILABLE` | 정규화 파라미터가 없거나 `period=7d`라 `hotScore`를 낼 수 없음. 추정값을 만들지 않고 null을 준다 | topics/{id}, search |

### 1.5 시각 표현

모든 시각은 KST ISO 8601이고 오프셋 `+09:00`을 붙인다.

| 종류 | 형식 | 예 |
| --- | --- | --- |
| 일시 | `yyyy-MM-dd'T'HH:mm:ss+09:00` | `2026-09-14T14:00:00+09:00` |
| 시간 버킷 | 위와 같되 분·초는 00 | `2026-09-14T13:00:00+09:00` |
| 일자 | `yyyy-MM-dd` | `2026-09-13` |

### 1.6 캐시

| 엔드포인트 | Cache-Control | 근거 |
| --- | --- | --- |
| `/api/v1/trends` | `public, max-age=<다음 게시 예상까지 초, 최소 30 최대 900>` | 스냅샷은 다음 게시까지 불변 |
| `/api/v1/topics/{topicId}` | `public, max-age=60` | |
| `/api/v1/topics/{topicId}/timeline` | `public, max-age=300` | 버킷 단위 데이터 |
| `/api/v1/topics/{topicId}/articles` | `public, max-age=120` | 런마다 늘어난다 |
| `/api/v1/topics/{topicId}/related-keywords` | `public, max-age=300` | |
| `/api/v1/search` | `public, max-age=60` | |
| `/health` | `no-store` | |
| 4xx, 5xx | `no-store` | |

`dataStatus`가 `stale`이면 `max-age`는 하한 30으로 고정해 프론트가 자주 재확인하게 한다.

### 1.7 공통 에러 형식

```json
{
  "code": "INVALID_CATEGORY",
  "message": "지원하지 않는 category 값입니다: economy",
  "timestamp": "2026-09-14T14:35:02+09:00"
}
```

| 필드 | 설명 |
| --- | --- |
| code | 기계가 분기할 상수 (8절) |
| message | 사람이 읽는 한국어 설명. 문구는 예고 없이 바뀔 수 있으므로 분기에는 code를 쓴다 |
| timestamp | 오류 발생 서버 시각 (KST) |

`message`에 스택 트레이스, SQL, 내부 클래스명을 넣지 않는다.

### 1.8 엔드포인트 요약

| 메서드 | 경로 | 목적 | 01 문서 대응 |
| --- | --- | --- | --- |
| GET | `/api/v1/trends?category=&period=&sort=&limit=` | 이슈 순위 | 기능 1 |
| GET | `/api/v1/topics/{topicId}?period=&snapshotId=` | 이슈 상세 | 기능 2, 3.2.1 |
| GET | `/api/v1/topics/{topicId}/timeline?period=&interval=` | 시간별 추이 | 기능 2, 3.2.2 |
| GET | `/api/v1/topics/{topicId}/articles?page=&size=` | 관련 기사 | 기능 2, 3.2.3 |
| GET | `/api/v1/topics/{topicId}/related-keywords` | 연관 키워드 | 기능 2, 3.2.4 |
| GET | `/api/v1/search?q=&limit=` | 이슈 검색 | 기능 3 |
| GET | `/health` | 운영 신호 | 02 문서 8.4절 |

정의서 §7의 사용자 흐름은 `trends` → `topics/{id}` → (`timeline`, `articles`, `related-keywords` 병렬) → 외부 링크 순으로 이 엔드포인트를 탄다.

---

## 2. GET /api/v1/trends

게시된 스냅샷에서 카테고리·기간·정렬 조합의 순위를 돌려준다.

### 2.1 요청

| 파라미터 | 필수 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- | --- |
| category | 아니오 | string | `ALL` | `ALL`, `ECONOMY`, `SPORTS`, `ENTERTAINMENT`, `POLITICS_SOCIAL`, `IT_SCIENCE` |
| period | 아니오 | string | `24h` | `1h`, `6h`, `24h`, `7d` |
| sort | 아니오 | string | `hot` | `hot`, `growth`, `mentions` |
| limit | 아니오 | int | 20 | 1~50 |

```http
GET /api/v1/trends?category=ECONOMY&period=1h&sort=growth&limit=20 HTTP/1.1
Host: api.koreapulse.example
Accept: application/json
```

### 2.2 응답 200

```json
{
  "meta": {
    "snapshotId": 90312,
    "generatedAt": "2026-09-14T14:03:20+09:00",
    "dataThrough": "2026-09-14T14:00:00+09:00",
    "period": "1h",
    "timezone": "Asia/Seoul",
    "dataStatus": "fresh",
    "scoreVersion": "hot-v0.2-exp1",
    "availablePeriods": ["1h", "6h", "24h"],
    "coveredCategories": ["ECONOMY", "SPORTS"],
    "notices": []
  },
  "category": "ECONOMY",
  "sort": "growth",
  "items": [
    {
      "rank": 1,
      "topicId": 48213,
      "title": "기준금리 동결 한국은행",
      "category": "ECONOMY",
      "mentionCount": 42,
      "rawArticleCount": 67,
      "previousMentionCount": 12,
      "growthRate": 250.0,
      "growthDelta": 30,
      "growthStatus": "OK",
      "sourceCount": 18,
      "hotScore": 88.4,
      "reason": "최근 1시간 42건, 이전 1시간 12건"
    },
    {
      "rank": 2,
      "topicId": 48377,
      "title": "코스피 3400 돌파",
      "category": "ECONOMY",
      "mentionCount": 12,
      "rawArticleCount": 15,
      "previousMentionCount": 0,
      "growthRate": null,
      "growthDelta": 12,
      "growthStatus": "NEW",
      "sourceCount": 9,
      "hotScore": 71.2,
      "reason": "최근 1시간 12건, 이전 1시간 0건"
    }
  ]
}
```

응답 헤더:

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Cache-Control: public, max-age=720
```

`growthRate`가 null인 항목은 `growthStatus`로 이유를 구분한다. `NEW`는 이전 구간 0건, `INSUFFICIENT`는 비교 구간 미확보나 **결손 비율 임계치(10%) 초과**다. `sort=growth`에서 `INSUFFICIENT` 항목은 제외되고, `NEW` 항목은 OK 항목 뒤에 `growthDelta` 내림차순으로 붙는다.

**정렬별 순위는 독립이다.** `sort=growth`의 1위가 `sort=hot`의 상위 50에 없어도 정상적으로 나온다. 세 정렬은 각각 전체 순위 후보를 대상으로 상위 50을 선정하며(01 문서 2.4절, 03 문서 3.13절), 따라서 동일 스냅샷에서도 sort에 따라 items 집합 자체가 다를 수 있다.

### 2.3 응답 필드

| 필드 | 타입 | nullable | 03 컬럼 | 설명 |
| --- | --- | --- | --- | --- |
| meta | object | 아니오 | trend_snapshot | 1.4절 |
| category, sort | string | 아니오 | | 적용된 요청 값 |
| items[].rank | int | 아니오 | trend_snapshot_entry.rank_hot / rank_growth / rank_mentions | 요청 `sort`에 대응하는 컸럼값. 정렬 3종은 각각 독립으로 상위 50을 선정한다 |
| items[].topicId | long | 아니오 | trend_snapshot_entry.topic_id | 상세 호출에 쓴다 |
| items[].title | string | 아니오 | topic.title | 대표 키워드 최대 3개 |
| items[].category | string | 아니오 | topic.category | 저장 코드 6종 중 하나. `UNCLASSIFIED`는 ALL 조회에서만 보인다 |
| items[].mentionCount | int | 아니오 | trend_snapshot_entry.mention_count | 현재 구간 중복 제거 기사 수 |
| items[].rawArticleCount | int | 아니오 | trend_snapshot_entry.raw_article_count | 중복 제거 전 기사 수 |
| items[].previousMentionCount | int | 아니오 | trend_snapshot_entry.previous_mention_count | 비교 구간 값 |
| items[].growthRate | float | 예 | trend_snapshot_entry.growth_rate | 퍼센트. `(현재-이전)/이전×100`. 이전 0건·비교 불가면 null |
| items[].growthDelta | int | 아니오 | trend_snapshot_entry.growth_delta | 현재 - 이전 |
| items[].growthStatus | string | 아니오 | trend_snapshot_entry.growth_status | `OK` / `NEW` / `INSUFFICIENT` |
| items[].sourceCount | int | 아니오 | trend_snapshot_entry.source_count | 현재 구간 출처 수 |
| items[].hotScore | float | 아니오 | trend_snapshot_entry.hot_score | 0~100 내부 상대 점수. 진실성·중요도가 아니다 |
| items[].reason | string | 아니오 | trend_snapshot_entry.reason | 실제 집계로 생성한 근거 문장 |

`/trends`에는 `searchInterest`가 **없다**. 검색 관심도는 순위·점수와 분리된 상세 전용 보조 지표라(01 문서 6.3절, 03 문서 3.13절) `/topics/{topicId}`에서만 내려간다. 순위 응답에 실으면 Redis 랭킹이 적중해도 항목 50건을 PostgreSQL에 조인해야 해 캐시가 무의미해진다.

`hotScore`의 증가율 입력은 평활 상수 k=3을 쓴 `(현재-이전)/(이전+3)`이고, 표시용 `growthRate`는 정의서 §8 원식이다. 두 값이 다른 이유는 01 문서 8.1절에 있다.

### 2.4 오류

| 상태 | code | 상황 |
| --- | --- | --- |
| 400 | `INVALID_CATEGORY` | 조회 코드 6종 밖의 값 |
| 400 | `INVALID_PARAMETER` | period·sort 값 오류, limit이 1~50 밖 |
| 503 | `NO_SNAPSHOT_AVAILABLE` | 게시된 스냅샷이 없음. `Retry-After: 300` 동봉 |

마지막 런이 실패한 경우는 오류가 아니다. 직전 게시 스냅샷을 200으로 주고 `meta.dataStatus`가 `stale`이 된다. 부분 수집 실패가 임계치 이하일 때도 오류가 아니며, 지표는 그대로 주고 `notices`에 `COVERAGE_DEGRADED`만 넣는다. 요청한 기간의 비교 구간이 확보되지 않은 경우도 오류가 아니다. `dataStatus`는 `insufficient`, `growthRate`는 전부 null이며, `sort=growth` 요청은 mentions 순서로 폴백하고 `notices`에 `SORT_FALLBACK_MENTIONS`가 들어간다.

---

## 3. GET /api/v1/topics/{topicId}

이슈 하나의 요약 지표, 근거, 대표 기사를 돌려준다.

### 3.1 요청

| 파라미터 | 위치 | 필수 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- | --- | --- |
| topicId | path | 예 | long | | `/trends`가 돌려준 값 |
| period | query | 아니오 | string | `24h` | 지표 계산 구간 |
| snapshotId | query | 아니오 | long | 최신 게시 | 순위와 같은 기준을 고정할 때 사용 |

### 3.2 응답 200

```json
{
  "meta": {
    "snapshotId": 90312,
    "generatedAt": "2026-09-14T14:03:20+09:00",
    "dataThrough": "2026-09-14T14:00:00+09:00",
    "period": "24h",
    "timezone": "Asia/Seoul",
    "dataStatus": "fresh",
    "scoreVersion": "hot-v0.2-exp1",
    "availablePeriods": ["1h", "6h", "24h"],
    "coveredCategories": ["ECONOMY", "SPORTS"],
    "notices": []
  },
  "topicId": 48213,
  "title": "기준금리 동결 한국은행",
  "category": "ECONOMY",
  "tags": ["통화정책", "금리"],
  "status": "ACTIVE",
  "rank": 1,
  "mentionCount": 137,
  "rawArticleCount": 212,
  "previousMentionCount": 54,
  "growthRate": 153.7,
  "growthDelta": 83,
  "growthStatus": "OK",
  "sourceCount": 41,
  "hotScore": 88.4,
  "reason": "최근 24시간 137건, 이전 24시간 54건",
  "firstSeenAt": "2026-09-13T09:31:05+09:00",
  "lastMentionedAt": "2026-09-14T13:52:00+09:00",
  "representativeArticles": [
    {
      "title": "한은, 기준금리 연 2.50%로 동결... 가계부채 증가세 경계",
      "originalUrl": "https://www.example-news.co.kr/economy/2026/09/14/0001234567",
      "providerUrl": "https://n.news.naver.com/mnews/article/001/0015123456",
      "source": "예시일보",
      "publishedAt": "2026-09-14T13:52:00+09:00",
      "duplicateCount": 6
    }
  ],
  "searchInterest": {
    "index": 87,
    "asOf": "2026-09-13",
    "keywordGroup": ["한국은행", "한은"],
    "periodFrom": "2026-09-07",
    "periodTo": "2026-09-13",
    "unavailableReason": null
  },
  "providerNotice": "출처: 네이버 뉴스"
}
```

검색 관심도가 없는 이슈는 `searchInterest: null`이다. `searchInterest`를 내려주는 엔드포인트는 **이곳뿐**이다.

순위에 오르지 못한 이슈는 `rank: null`이고 구간 지표는 조회 시점에 같은 규칙으로 계산한다. 단 `hotScore`는 스냅샷 헤더의 정규화 파라미터가 있을 때만 재현하고, 없거나 `period=7d`면 `null` + `notices`에 `SCORE_UNAVAILABLE`을 넣는다(01 문서 3.2.1절). 추정값을 만들지 않는다.

### 3.3 응답 필드

| 필드 | 타입 | nullable | 03 컬럼 | 설명 |
| --- | --- | --- | --- | --- |
| topicId | long | 아니오 | topic.id | |
| title | string | 아니오 | topic.title | |
| category | string | 아니오 | topic.category | |
| tags | string[] | 아니오 | topic_tag.tag | 보조 태그. 없으면 빈 배열 |
| status | string | 아니오 | topic.status | `ACTIVE` / `DORMANT` |
| rank | int | 예 | trend_snapshot_entry.rank_hot | 해당 스냅샷·기간의 ALL 카테고리 hot 순위. 순위권 밖이면 null |
| mentionCount ~ reason | | | trend_snapshot_entry | 2.3절과 동일 정의. 단 `hotScore`는 **nullable**이다 (위 설명) |
| firstSeenAt | datetime | 아니오 | topic.first_seen_at | |
| lastMentionedAt | datetime | 아니오 | topic.last_mentioned_at | |
| representativeArticles | array (0~3) | 아니오 | article (중복 그룹 대표) | 출처 다양성·최신순 |
| searchInterest | object | 예 | search_interest_snapshot | 6절 |
| providerNotice | string | 아니오 | (설정값) | 공급자별 출처 표기 문자열 |

AI 요약 필드(`summary`, `summaryGeneratedAt`)는 MVP 응답에 없다(01 문서 1.2절).

### 3.4 오류

| 상태 | code | 상황 |
| --- | --- | --- |
| 400 | `INVALID_PARAMETER` | topicId가 양의 정수가 아님, period 값 오류 |
| 404 | `TOPIC_NOT_FOUND` | 해당 id의 이슈 없음 |

`DORMANT` 이슈는 404가 아니다. 지정한 `snapshotId`가 없거나 만료됐으면 최신 게시 스냅샷으로 폴백하고 `meta.notices`에 `SNAPSHOT_FALLBACK`을 넣는다.

---

## 4. GET /api/v1/topics/{topicId}/timeline

이슈의 시간별 추이를 돌려준다. 누락 구간과 실제 0을 구분한다.

### 4.1 요청

| 파라미터 | 위치 | 필수 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- | --- | --- |
| topicId | path | 예 | long | | |
| period | query | 아니오 | string | `24h` | `1h`, `6h`, `24h`, `7d` |
| interval | query | 아니오 | string | period에 따름 | `hour` 또는 `day`. 1h·6h·24h는 hour, 7d는 day |
| from, to | query | 아니오 | datetime | period로 계산 | 직접 지정 시 반개구간 `[from, to)`, 최대 폭 14일 |

`from`·`to`는 오프셋을 포함한 ISO 8601이면 어떤 시간대로 보내도 되고, 생략하면 KST로 해석한다. URL에서 `+`는 `%2B`로 인코딩한다.

### 4.2 응답 200

```json
{
  "meta": {
    "snapshotId": 90312,
    "generatedAt": "2026-09-14T14:03:20+09:00",
    "dataThrough": "2026-09-14T14:00:00+09:00",
    "period": "6h",
    "timezone": "Asia/Seoul",
    "dataStatus": "fresh",
    "scoreVersion": "hot-v0.2-exp1",
    "availablePeriods": ["1h", "6h", "24h"],
    "coveredCategories": ["ECONOMY", "SPORTS"],
    "notices": []
  },
  "topicId": 48213,
  "interval": "hour",
  "from": "2026-09-14T08:00:00+09:00",
  "to": "2026-09-14T14:00:00+09:00",
  "points": [
    { "bucket": "2026-09-14T08:00:00+09:00", "mentionCount": 18, "status": "OK" },
    { "bucket": "2026-09-14T09:00:00+09:00", "mentionCount": 22, "status": "OK" },
    { "bucket": "2026-09-14T10:00:00+09:00", "mentionCount": 9, "status": "PARTIAL" },
    { "bucket": "2026-09-14T11:00:00+09:00", "mentionCount": 0, "status": "MISSING" },
    { "bucket": "2026-09-14T12:00:00+09:00", "mentionCount": 29, "status": "OK" },
    { "bucket": "2026-09-14T13:00:00+09:00", "mentionCount": 42, "status": "OK" }
  ]
}
```

### 4.3 응답 필드

| 필드 | 타입 | nullable | 03 컬럼 | 설명 |
| --- | --- | --- | --- | --- |
| interval | string | 아니오 | | 적용된 간격 |
| from, to | datetime | 아니오 | | 실제 적용된 반개구간 |
| points[].bucket | datetime | 아니오 | topic_hourly_stat.bucket_hour | 구간 시작 시각 |
| points[].mentionCount | int | 아니오 | topic_hourly_stat.mention_count | 중복 제거 기사 수 |
| points[].status | string | 아니오 | bucket_coverage.status | `OK` / `PARTIAL` / `MISSING` |

`status`가 OK가 아닌 점의 `mentionCount`는 참고값이다. 프론트는 실제 0(`OK` + 0건)과 수집 실패(`PARTIAL`, `MISSING`)를 다르게 그린다(정의서 §7). `interval=day`이면 KST 날짜 경계로 버킷을 합치고, 하루 안에 OK가 아닌 버킷이 있으면 그 날의 status는 `PARTIAL`이다.

### 4.4 오류

| 상태 | code | 상황 |
| --- | --- | --- |
| 400 | `INVALID_PARAMETER` | topicId 형식 오류, period·interval 값 오류, from > to, 폭 14일 초과 |
| 404 | `TOPIC_NOT_FOUND` | 해당 id의 이슈 없음 |

---

## 5. GET /api/v1/topics/{topicId}/articles

이슈에 연결된 기사를 중복 그룹 단위로, 발행 시각 내림차순 페이징해 돌려준다.

본문과 스니펫은 응답에 없다. 제목과 링크만 주고 읽기는 원문 페이지에서 이루어진다(01 문서 9절). 출처 표기는 서버가 `providerNotice`로 실어 누락을 막는다.

### 5.1 요청

| 파라미터 | 위치 | 필수 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- | --- | --- |
| topicId | path | 예 | long | | |
| page | query | 아니오 | int | 0 | 0부터 시작 |
| size | query | 아니오 | int | 10 | 1~50 |

### 5.2 응답 200

```json
{
  "meta": {
    "snapshotId": null,
    "generatedAt": null,
    "dataThrough": "2026-09-14T14:00:00+09:00",
    "period": null,
    "timezone": "Asia/Seoul",
    "dataStatus": "fresh",
    "scoreVersion": null,
    "availablePeriods": ["1h", "6h", "24h"],
    "coveredCategories": ["ECONOMY", "SPORTS"],
    "notices": []
  },
  "topicId": 48213,
  "totalCount": 137,
  "page": 0,
  "size": 10,
  "partial": false,
  "providerNotice": "출처: 네이버 뉴스",
  "items": [
    {
      "title": "한은, 기준금리 연 2.50%로 동결... 가계부채 증가세 경계",
      "originalUrl": "https://www.example-news.co.kr/economy/2026/09/14/0001234567",
      "providerUrl": "https://n.news.naver.com/mnews/article/001/0015123456",
      "source": "예시일보",
      "publishedAt": "2026-09-14T13:52:00+09:00",
      "duplicateCount": 6
    },
    {
      "title": "[속보] 한국은행 기준금리 동결 결정",
      "originalUrl": "https://news.example-broadcast.kr/view/20260914135505",
      "providerUrl": null,
      "source": null,
      "publishedAt": "2026-09-14T13:55:05+09:00",
      "duplicateCount": 1
    }
  ]
}
```

### 5.3 응답 필드

| 필드 | 타입 | nullable | 03 컬럼 | 설명 |
| --- | --- | --- | --- | --- |
| totalCount | int | 아니오 | count(중복 그룹) | 보존 기간 만료로 삭제된 기사는 빠진다 |
| page, size | int | 아니오 | | 적용된 값 |
| partial | boolean | 아니오 | | 외부 공급자 보강 호출이 실패해 저장분만으로 응답했으면 true |
| providerNotice | string | 아니오 | (설정값) | 공급자별 출처 표기 |
| items[].title | string | 아니오 | article.title | 태그 제거 후 제목 |
| items[].originalUrl | string | 아니오 | article.original_url | 언론사 원문 URL |
| items[].providerUrl | string | 예 | article.provider_url | 공급자 링크. 없으면 null |
| items[].source | string | 예 | article.source_name | 매체 표시명. 매핑이 없으면 null (추정 금지) |
| items[].publishedAt | datetime | 아니오 | article.published_at | |
| items[].duplicateCount | int | 아니오 | article_duplicate_group.member_count | 이 그룹에 묶인 기사 수 |

관련 기사 조회 실패가 상세 화면 전체를 실패시키지 않는다(정의서 §10). 외부 보강이 실패하면 `partial: true`와 `notices`에 `PROVIDER_PARTIAL`을 넣고 200으로 응답한다.

### 5.4 오류

| 상태 | code | 상황 |
| --- | --- | --- |
| 400 | `INVALID_PARAMETER` | topicId 형식 오류, page 음수, size가 1~50 밖 |
| 404 | `TOPIC_NOT_FOUND` | 해당 id의 이슈 없음 |

범위를 넘는 `page`는 오류가 아니다. `items`가 빈 배열이고 `totalCount`는 정상 값이다.

---

## 6. GET /api/v1/topics/{topicId}/related-keywords

```json
{
  "topicId": 48213,
  "items": [
    { "keyword": "기준금리", "articleCount": 121 },
    { "keyword": "한국은행", "articleCount": 118 },
    { "keyword": "금통위", "articleCount": 64 }
  ]
}
```

| 필드 | 타입 | 03 컬럼 | 설명 |
| --- | --- | --- | --- |
| items[].keyword | string | topic_keyword.keyword | 최대 10개 |
| items[].articleCount | int | topic_keyword.article_count | 내림차순 정렬 기준 |

오류는 `INVALID_PARAMETER`(400), `TOPIC_NOT_FOUND`(404)다.

### 6.1 searchInterest 객체

`/topics/{topicId}` 상세 응답 **전용** 객체다. `/trends`와 `/search`에는 들어가지 않는다(01 문서 6.3절).

| 필드 | 타입 | nullable | 03 컬럼 | 설명 |
| --- | --- | --- | --- | --- |
| index | int (0~100) | 아니오 | search_interest_snapshot.ratio | 상대 지수를 반올림한 정수 |
| asOf | date | 아니오 | search_interest_snapshot.stat_date | 기준 일자. 보통 전일 |
| keywordGroup | string[] | 아니오 | search_interest_snapshot.keyword_group | 조회에 사용한 키워드 그룹 |
| periodFrom, periodTo | date | 아니오 | period_from, period_to | 조회 기간 |
| unavailableReason | string | 예 | unavailable_reason | 값이 없을 때만 채워지는 사유 |

객체 전체가 null이면 검색 데이터가 없다는 뜻이다. 프론트는 "검색 데이터 없음"으로 표시하고, index를 `asOf` 없이 단독 표시하지 않는다. 이 값은 일 단위 상대 지수이며 실시간 검색량이 아니다.

---

## 7. GET /api/v1/search

추적 중인 이슈를 제목·별칭·연관 키워드로 찾는다.

### 7.1 요청

| 파라미터 | 필수 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- | --- |
| q | 예 | string | | 트림 후 1~50자 |
| limit | 아니오 | int | 20 | 1~50 |

```http
GET /api/v1/search?q=%EC%82%BC%EC%84%B1%EC%A0%84%EC%9E%90 HTTP/1.1
```

### 7.2 응답 200

```json
{
  "meta": {
    "snapshotId": 90312,
    "generatedAt": "2026-09-14T14:03:20+09:00",
    "dataThrough": "2026-09-14T14:00:00+09:00",
    "period": "24h",
    "timezone": "Asia/Seoul",
    "dataStatus": "fresh",
    "scoreVersion": "hot-v0.2-exp1",
    "availablePeriods": ["1h", "6h", "24h"],
    "coveredCategories": ["ECONOMY", "SPORTS"],
    "notices": []
  },
  "query": "삼성전자",
  "totalCount": 2,
  "emptyReason": null,
  "items": [
    {
      "topicId": 48555,
      "title": "삼성전자 HBM 공급 계약",
      "category": "IT_SCIENCE",
      "matchedBy": "KEYWORD",
      "mentionCount": 61,
      "hotScore": 74.9,
      "lastMentionedAt": "2026-09-14T13:40:00+09:00"
    },
    {
      "topicId": 48501,
      "title": "삼성전자 3분기 실적 발표",
      "category": "ECONOMY",
      "matchedBy": "TITLE",
      "mentionCount": 44,
      "hotScore": 68.1,
      "lastMentionedAt": "2026-09-14T12:10:00+09:00"
    }
  ]
}
```

매칭 이슈가 없을 때:

```json
{
  "meta": { "...": "생략" },
  "query": "없는키워드",
  "totalCount": 0,
  "emptyReason": "NOT_TRACKED",
  "items": []
}
```

### 7.3 응답 필드

| 필드 | 타입 | nullable | 출처 | 설명 |
| --- | --- | --- | --- | --- |
| query | string | 아니오 | | 정규화 전 입력값 |
| totalCount | int | 아니오 | | 반환 항목 수 (최대 limit) |
| emptyReason | string | 예 | | 결과가 없을 때 `NOT_TRACKED` |
| items[].matchedBy | string | 아니오 | | `TITLE` / `ALIAS` / `KEYWORD` |
| items[].mentionCount | int | 아니오 | 24h 구간 집계 | 정렬은 hotScore 내림차순 |
| items[].hotScore | float | **예** | trend_snapshot_entry 또는 조회 시점 재현 | 추적 중인 이슈의 값만 준다. 정규화 파라미터가 없으면 null + `SCORE_UNAVAILABLE` |

검색되지 않은 키워드의 Hot Score를 만들어 내지 않는다(정의서 §7). 미추적 키워드는 빈 배열과 `NOT_TRACKED`를 준다.

### 7.4 오류

| 상태 | code | 상황 |
| --- | --- | --- |
| 400 | `INVALID_PARAMETER` | q 누락·공백만·50자 초과, limit 범위 밖 |

---

## 8. GET /health

```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "redis": { "status": "UP" },
    "pipeline": {
      "status": "UP",
      "details": {
        "lastSuccessAt": "2026-09-14T14:03:20+09:00",
        "minutesSinceLastSuccess": 4,
        "lastRunStatus": "SUCCESS",
        "cadenceMinutes": 15,
        "currentSnapshotId": 90312,
        "ingestLagMinutes": 11
      }
    }
  }
}
```

| 필드 | 타입 | nullable | 03 컬럼 | 설명 |
| --- | --- | --- | --- | --- |
| components.db.status | string | 아니오 | | 실패 시 전체 status도 DOWN |
| components.redis.status | string | 아니오 | | Redis 비활성·장애면 `DOWN`이지만 전체 status는 UP을 유지한다 (PostgreSQL 폴백) |
| pipeline.details.lastSuccessAt | datetime | 예 | collection_run.finished_at | 마지막 SUCCESS 런 |
| pipeline.details.minutesSinceLastSuccess | int | 예 | | 없으면 null |
| pipeline.details.lastRunStatus | string | 예 | collection_run.status | |
| pipeline.details.cadenceMinutes | int | 아니오 | collection_run.cadence_minutes | 15 / 30 / 60 |
| pipeline.details.currentSnapshotId | long | 예 | trend_snapshot.id (PUBLISHED) | |
| pipeline.details.ingestLagMinutes | int | 예 | collection_run.ingest_lag_seconds | 뉴스 발행 → 화면 노출 지연 (정의서 §9) |

DB 연결이 끊긴 경우에만 503이며 Actuator 형식으로 응답한다. 파이프라인 지연은 재시작으로 고쳐지지 않으므로 DOWN으로 내리지 않는다.

---

## 9. 에러 코드

| 상태 | code | 발생 엔드포인트 | 상황 |
| --- | --- | --- | --- |
| 400 | `INVALID_CATEGORY` | trends | 조회 코드 6종 밖의 category |
| 400 | `INVALID_PARAMETER` | 전체 | period·sort·interval 값 오류, id 형식 오류, from > to, 폭 14일 초과, page·size·limit 범위 밖, q 누락 |
| 404 | `TOPIC_NOT_FOUND` | topics/* | 존재하지 않는 이슈 |
| 429 | `RATE_LIMITED` | 전체 (`/health` 제외) | 같은 IP 분당 한도(기본 60회) 초과. `Retry-After` 동봉 |
| 500 | `INTERNAL_ERROR` | 전체 | 처리되지 않은 예외 |
| 503 | `NO_SNAPSHOT_AVAILABLE` | trends | 게시된 스냅샷이 한 번도 없음. `Retry-After: 300` 동봉 |

`topics/*`와 `search`는 게시된 스냅샷이 없어도 503을 내지 않는다. 이슈는 스냅샷과 무관하게 존재할 수 있으므로 있으면 200, 없으면 404다.

### 9.1 예시 본문

```json
{
  "code": "INVALID_PARAMETER",
  "message": "지원하지 않는 period 값입니다: 12h. 허용값: 1h, 6h, 24h, 7d",
  "timestamp": "2026-09-14T14:36:11+09:00"
}
```

```json
{
  "code": "NO_SNAPSHOT_AVAILABLE",
  "message": "아직 집계된 데이터가 없습니다. 잠시 후 다시 시도해 주세요",
  "timestamp": "2026-09-14T14:39:03+09:00"
}
```

### 9.2 오류 응답 헤더

| 상태 | 헤더 |
| --- | --- |
| 모든 4xx, 5xx | `Cache-Control: no-store`, `Content-Type: application/json; charset=utf-8` |
| 503 | 위 + `Retry-After: 300` |

---

## 10. 다른 문서와의 대응

| 이 문서 | 대응 |
| --- | --- |
| 1.4절 공통 메타 | 정의서 §10, 01 문서 2.2·2.5절 |
| 2절 `/trends` | 01 문서 기능 1, 03 문서 3.12·3.13 |
| 3절 `/topics/{id}` | 01 문서 3.2.1, 03 문서 3.6·3.7 |
| 4절 `/timeline` | 01 문서 3.2.2, 03 문서 3.10·3.11 |
| 5절 `/articles` | 01 문서 3.2.3과 9절, 03 문서 3.2·3.3 |
| 6절 `/related-keywords`, searchInterest | 01 문서 3.2.4·6절, 03 문서 3.8·3.14 |
| 7절 `/search` | 01 문서 기능 3, 03 문서 3.5 |
| 8절 `/health` | 02 문서 8.4절, 03 문서 3.1 |
| 1.6절 캐시 | 02 문서 4장 Redis 랭킹, 9장 성능 가정 |
