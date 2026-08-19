# 알림 시스템 (Notification System) API 명세서

> **문서 목적**: 발신 서비스가 알림 시스템과 통신하기 위한 API 인터페이스를 정의한다.
> **참조 문서**: [`01-notification-requirements.md`](01-notification-requirements.md), [`02-notification-architecture.md`](02-notification-architecture.md), [`03-notification-data-model.md`](03-notification-data-model.md)
> **통신 방식**: HTTP/1.1 REST API
> **기본 URL**: `https://api.example.com/v1`

---

## 1. 공통 규약

### 1.1 요청 / 응답 형식

- 요청 본문: `application/json`
- 응답 본문: `application/json`
- 문자 인코딩: `UTF-8`
- 시각 표기: **ISO 8601 UTC** (`2026-08-14T09:30:00Z`)

### 1.2 인증 — appKey / appSecret (HMAC 서명)

> **알림 전송 API는 스팸 방지를 위해 사내 서비스 또는 인증된 클라이언트만 이용 가능하다.** (NFR-6)

발신 서비스는 발급받은 `appKey`(공개)와 `appSecret`(비공개)을 사용해 **요청에 서명**한다.
`appSecret`은 **절대 전송하지 않는다.**

```text
signing_string = HTTP_METHOD + "\n"
               + PATH + "\n"
               + X-Timestamp + "\n"
               + SHA256_HEX(request_body)

X-Signature = HMAC_SHA256(appSecret, signing_string)  →  hex
```

| 헤더 | 필수 | 설명 | 예시 |
| --- | :---: | --- | --- |
| `X-App-Key` | ✅ | 발신 서비스 식별자 | `svc-billing-prod` |
| `X-Timestamp` | ✅ | 요청 생성 시각(ISO 8601 UTC) | `2026-08-14T09:30:00Z` |
| `X-Signature` | ✅ | 위 규칙으로 생성한 HMAC-SHA256 | `9f86d081884c7d65...` |
| `X-Request-Id` | 선택 | 요청 추적용 ID (미지정 시 서버 생성) | `req-550e8400-e29b` |
| `Content-Type` | ✅ | | `application/json` |

**서버 검증 순서**
1. `X-App-Key`가 유효하고 활성 상태인가
2. `X-Timestamp`가 **현재 시각 ±5분** 이내인가 (재전송 공격 방지)
3. `X-Signature`가 일치하는가
4. 해당 appKey가 요청한 **채널·템플릿 사용 권한**을 갖는가

> ⚠️ 실패는 전부 `401 UNAUTHORIZED`로 통일한다. 어느 단계에서 틀렸는지 알려주지 않는다.

### 1.3 멱등성 — `event_id`

발신 서비스는 요청마다 **고유한 `event_id`** 를 부여해야 한다.

| 상황 | 서버 동작 |
| --- | --- |
| 처음 보는 `event_id` | 정상 처리, `202 Accepted` |
| **24시간 내 재사용된 `event_id`** | 새로 발송하지 않고 **기존 결과를 그대로 반환** (`200 OK`, `idempotent_replay: true`) |

> 이것이 [정리노트 STEP 5](../정리노트/05_STEP5_안정성_데이터손실_중복전송.md)의 **중복 방지 로직 1단계**다.
> 발신 서비스가 타임아웃 후 안전하게 재시도할 수 있게 만드는 계약이기도 하다.

### 1.4 공통 응답 헤더

| 헤더 | 설명 |
| --- | --- |
| `X-Request-Id` | 요청 추적 ID |
| `X-RateLimit-Limit` | 해당 appKey의 시간당 호출 허용량 |
| `X-RateLimit-Remaining` | 잔여 호출 수 |
| `X-RateLimit-Reset` | 윈도우 초기화 시각(Unix epoch) |

### 1.5 공통 에러 응답 형식

```json
{
  "error": {
    "code": "INVALID_TEMPLATE_PARAMS",
    "message": "Required parameter 'item_name' is missing.",
    "request_id": "req-550e8400-e29b",
    "details": [
      { "field": "template.params.item_name", "reason": "required" }
    ]
  }
}
```

### 1.6 에러 코드 목록

| HTTP | `error.code` | 설명 |
| :---: | --- | --- |
| `400` | `INVALID_REQUEST` | 필수 필드 누락 또는 형식 오류 |
| `400` | `INVALID_CHANNEL` | 지원하지 않는 채널 |
| `400` | `INVALID_TEMPLATE_PARAMS` | 템플릿 필수 인자 누락 |
| `400` | `TEMPLATE_RENDER_FAILED` | 템플릿 렌더링 실패 |
| `400` | `PAYLOAD_TOO_LARGE` | 본문이 채널 상한 초과 |
| `401` | `UNAUTHORIZED` | 인증 실패 (키/서명/타임스탬프) |
| `403` | `CHANNEL_NOT_ALLOWED` | 해당 appKey에 채널·템플릿 권한 없음 |
| `404` | `USER_NOT_FOUND` | 대상 사용자 없음 |
| `404` | `TEMPLATE_NOT_FOUND` | 템플릿 없음 또는 비활성 |
| `404` | `NOTIFICATION_NOT_FOUND` | 조회한 알림 ID 없음 |
| `409` | `DUPLICATE_EVENT_ID` | (엄격 모드에서) 중복 event_id 거부 |
| `422` | `NO_DELIVERABLE_TARGET` | 활성 단말·연락처가 하나도 없음 |
| `429` | `APP_RATE_LIMITED` | 발신 서비스 호출 한도 초과 |
| `500` | `INTERNAL_ERROR` | 서버 내부 오류 |
| `503` | `LOG_STORE_UNAVAILABLE` | **알림 로그 저장 불가 → 무손실 보장 불가하므로 접수 거부** |
| `503` | `QUEUE_UNAVAILABLE` | 큐 장애 |

> ⚠️ `503 LOG_STORE_UNAVAILABLE`은 의도적인 **fail-close**다. 받아놓고 잃어버리느니 명확히 거부한다. ([02 아키텍처 §6](02-notification-architecture.md))

### 1.7 제한값

| 항목 | 상한 |
| --- | ---: |
| 단건 요청 수신자 수 | 100 |
| 배치 요청 항목 수 | 1,000 |
| 요청 본문 크기 | 1 MB |
| 푸시 페이로드 | 4 KB (APNS 제약) |
| SMS 본문 | 1,000자 |
| 이메일 본문 | 256 KB |

---

## 2. API 목록

| 메서드 | 경로 | 설명 |
| --- | --- | --- |
| `POST` | `/v1/notifications` | **알림 발송 요청** (핵심 API) |
| `POST` | `/v1/notifications/batch` | 대량 발송 요청 |
| `GET` | `/v1/notifications/{notification_id}` | 알림 상태 조회 |
| `GET` | `/v1/notifications` | 알림 목록 조회 (필터) |
| `GET` | `/v1/users/{user_id}/notification-settings` | 수신 설정 조회 |
| `PUT` | `/v1/users/{user_id}/notification-settings` | 수신 설정 변경 |
| `POST` | `/v1/users/{user_id}/devices` | 단말 등록 |
| `DELETE` | `/v1/users/{user_id}/devices/{device_token}` | 단말 해제 |
| `POST` | `/v1/templates` | 템플릿 등록 |
| `GET` | `/v1/templates/{template_id}` | 템플릿 조회 |
| `POST` | `/v1/webhooks/delivery` | 제3자 전달 결과 수신 (인바운드) |
| `GET` | `/v1/health` | 상태 확인 |

---

## 3. 발송 API 상세

### 3.1 `POST /v1/notifications` — 알림 발송 요청

호출자는 **`user_id`와 템플릿**만 지정한다. 단말 토큰·전화번호·이메일 주소는 시스템이 조회한다.

#### 요청

```http
POST /v1/notifications HTTP/1.1
Host: api.example.com
Content-Type: application/json
X-App-Key: svc-commerce-prod
X-Timestamp: 2026-08-14T09:30:00Z
X-Signature: 9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
X-Request-Id: req-550e8400-e29b
```

```json
{
  "event_id": "order-88213-restock-notify",
  "channels": ["PUSH", "EMAIL"],
  "to": [
    { "user_id": 123456 }
  ],
  "template": {
    "id": "ITEM_RESTOCK",
    "version": 3,
    "params": {
      "item_name": "무선 이어폰",
      "date": "2026-08-20"
    }
  },
  "category": "MARKETING",
  "priority": "NORMAL",
  "scheduled_at": null,
  "override": {
    "from": { "email": "no-reply@example.com", "name": "Example Shop" }
  }
}
```

#### 요청 필드

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | :---: | --- |
| `event_id` | string | ✅ | **멱등 키.** 발신 서비스가 생성하는 전역 고유값 |
| `channels` | string[] | ✅ | `PUSH` / `SMS` / `EMAIL` 중 1개 이상 |
| `to` | object[] | ✅ | 수신자 목록. `{ "user_id": ... }` 형태, 최대 100건 |
| `template.id` | string | ✅ | 템플릿 ID |
| `template.version` | int | | 생략 시 최신 활성 버전 |
| `template.params` | object | ✅* | 템플릿의 `required_params` 전부 포함해야 함 |
| `category` | string | | `MARKETING` / `TRANSACTIONAL` / `SECURITY`. 기본 `TRANSACTIONAL` |
| `priority` | string | | `HIGH` / `NORMAL` / `LOW`. 기본 `NORMAL` |
| `scheduled_at` | string | | 예약 발송 시각(ISO 8601). `null`이면 즉시 |
| `override.from` | object | | 이메일 발신자 재정의 |

#### 응답 — `202 Accepted`

```json
{
  "request_id": "req-550e8400-e29b",
  "accepted_at": "2026-08-14T09:30:00Z",
  "idempotent_replay": false,
  "results": [
    {
      "user_id": 123456,
      "notifications": [
        {
          "notification_id": "3f2b9a10-6c1e-4b8d-9a77-1c0b8e5d4a21",
          "channel": "PUSH",
          "platform": "IOS",
          "status": "QUEUED"
        },
        {
          "notification_id": "8a1c4e32-2d55-49f0-b1a3-77e9c2f01b64",
          "channel": "PUSH",
          "platform": "ANDROID",
          "status": "QUEUED"
        },
        {
          "notification_id": "c74d1e88-9b02-4a3e-8f16-5d2a7c9b3e10",
          "channel": "EMAIL",
          "status": "QUEUED"
        }
      ]
    }
  ],
  "summary": { "queued": 3, "dropped": 0, "throttled": 0 }
}
```

> ⚠️ **1건의 요청이 3개의 `notification_id`를 만들었다.** 사용자가 iOS·Android 단말을 각각 보유했기 때문이다.
> 이것이 [FR-2 팬아웃](01-notification-requirements.md)이며, **추적·재시도의 단위는 `notification_id`** 다.

#### 부분 드롭 응답 (수신 거부된 채널이 있는 경우)

```json
{
  "results": [
    {
      "user_id": 123456,
      "notifications": [
        { "notification_id": "3f2b...", "channel": "PUSH", "status": "QUEUED" },
        { "notification_id": "c74d...", "channel": "EMAIL",
          "status": "DROPPED_OPT_OUT",
          "reason": "User opted out of EMAIL/MARKETING" }
      ]
    }
  ],
  "summary": { "queued": 1, "dropped": 1, "throttled": 0 }
}
```

> **설계 판단**: 수신 거부는 **에러가 아니라 정상 결과**다. `202`를 유지하고 상태로 표현한다.
> 발신 서비스가 이를 실패로 오인해 재시도하면 사용자 의사를 무시하는 결과가 되기 때문이다.

#### 상태 코드 요약

| 코드 | 상황 |
| :---: | --- |
| `202` | 접수 완료 (전부 또는 일부 큐 적재) |
| `200` | **멱등 재생** — 같은 `event_id`의 기존 결과 반환 (`idempotent_replay: true`) |
| `400` / `401` / `403` / `404` / `422` / `429` | §1.6 참조 |
| `503` | 로그 저장소·큐 장애로 접수 불가 |

---

### 3.2 `POST /v1/notifications/batch` — 대량 발송

마케팅 캠페인처럼 **수천~수만 명**에게 같은 템플릿을 보낼 때 사용한다.

#### 요청

```json
{
  "event_id": "campaign-2026-summer-sale-batch-0007",
  "channels": ["PUSH"],
  "category": "MARKETING",
  "template": { "id": "SUMMER_SALE", "version": 1 },
  "items": [
    { "user_id": 123456, "params": { "item_name": "무선 이어폰" } },
    { "user_id": 223344, "params": { "item_name": "블루투스 스피커" } }
  ]
}
```

#### 응답 — `202 Accepted`

```json
{
  "batch_id": "b-9c81f2a0-77de-4f10-9e2a-3b6c1d5e8a44",
  "accepted": 2,
  "rejected": 0,
  "status_url": "/v1/notifications?batch_id=b-9c81f2a0-77de-4f10-9e2a-3b6c1d5e8a44"
}
```

| 단건 API와의 차이 | 설명 |
| --- | --- |
| 개별 `notification_id`를 **즉시 반환하지 않는다** | 최대 1,000건의 팬아웃을 동기 구간에서 처리하면 P99 목표를 못 지킴 |
| `batch_id`로 **비동기 조회** | 접수만 확약하고 상세는 `status_url`로 |
| 항목별 실패 허용 | 일부 사용자 없음 → `rejected` 카운트에 반영, 전체는 진행 |

---

## 4. 조회 API

### 4.1 `GET /v1/notifications/{notification_id}` — 상태 조회

#### 응답 — `200 OK`

```json
{
  "notification_id": "3f2b9a10-6c1e-4b8d-9a77-1c0b8e5d4a21",
  "event_id": "order-88213-restock-notify",
  "user_id": 123456,
  "channel": "PUSH",
  "platform": "IOS",
  "provider": "APNS",
  "target": "a1b2****9f8e",
  "template": { "id": "ITEM_RESTOCK", "version": 3 },
  "rendered": {
    "title": "지금 무선 이어폰을 주문 또는 예약하세요!",
    "body": "여러분이 꿈꿔온 그 상품을 우리가 준비했습니다. 무선 이어폰이 다시 입고되었습니다! 2026-08-20까지만 주문 가능합니다!"
  },
  "status": "DELIVERED",
  "retry_count": 1,
  "last_error_code": null,
  "timeline": [
    { "event_type": "CREATED",   "occurred_at": "2026-08-14T09:30:00Z" },
    { "event_type": "QUEUED",    "occurred_at": "2026-08-14T09:30:00Z" },
    { "event_type": "SENDING",   "occurred_at": "2026-08-14T09:30:01Z" },
    { "event_type": "RETRYING",  "occurred_at": "2026-08-14T09:30:01Z", "detail": "APNS 503" },
    { "event_type": "SENT",      "occurred_at": "2026-08-14T09:30:06Z" },
    { "event_type": "DELIVERED", "occurred_at": "2026-08-14T09:30:07Z" }
  ],
  "created_at": "2026-08-14T09:30:00Z",
  "updated_at": "2026-08-14T09:30:07Z"
}
```

> `target`은 **마스킹**되어 반환된다. 원본 토큰·전화번호는 API로 절대 노출하지 않는다.

### 4.2 `GET /v1/notifications` — 목록 조회

| 쿼리 파라미터 | 설명 |
| --- | --- |
| `user_id` | 사용자별 조회 (`from`/`to`와 함께 사용 권장) |
| `batch_id` | 배치 단위 조회 |
| `event_id` | 멱등 키로 역조회 |
| `channel` | `PUSH` / `SMS` / `EMAIL` |
| `status` | 상태 필터 |
| `from`, `to` | 기간 (기본 최근 24시간, 최대 30일) |
| `cursor`, `limit` | 커서 페이지네이션 (기본 50, 최대 200) |

```json
{
  "items": [ { "notification_id": "...", "channel": "PUSH", "status": "SENT", "created_at": "..." } ],
  "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wOC0xNFQwOTowMDowMFoifQ==",
  "has_more": true
}
```

> ⚠️ **오프셋 페이지네이션을 쓰지 않는다.** 로그 저장소가 Cassandra이고 조회 키가 `(user_id, bucket) + created_at DESC`이므로 **커서 방식만 효율적**이다. ([03 데이터 모델 §3.2](03-notification-data-model.md))

---

## 5. 수신 설정 API

### 5.1 `GET /v1/users/{user_id}/notification-settings`

```json
{
  "user_id": 123456,
  "settings": [
    { "channel": "PUSH",  "category": "ALL",           "opt_in": true },
    { "channel": "PUSH",  "category": "MARKETING",     "opt_in": false },
    { "channel": "EMAIL", "category": "ALL",           "opt_in": true },
    { "channel": "SMS",   "category": "ALL",           "opt_in": false }
  ],
  "effective": {
    "PUSH":  { "MARKETING": false, "TRANSACTIONAL": true,  "SECURITY": true },
    "EMAIL": { "MARKETING": true,  "TRANSACTIONAL": true,  "SECURITY": true },
    "SMS":   { "MARKETING": false, "TRANSACTIONAL": false, "SECURITY": true }
  }
}
```

> `effective`는 [03 데이터 모델 §2.3의 우선순위 규칙](03-notification-data-model.md)을 적용한 **최종 판정 결과**다.
> 클라이언트가 규칙을 재구현하지 않아도 되게 서버가 계산해 준다.
> ⚠️ `SECURITY`는 어떤 설정에서도 `true`다 — 보안 통지는 거부 대상이 아니다.

### 5.2 `PUT /v1/users/{user_id}/notification-settings`

```json
{
  "settings": [
    { "channel": "PUSH", "category": "MARKETING", "opt_in": false }
  ]
}
```

**응답 `200 OK`** — 변경 즉시 **`setting:{user_id}` 캐시를 무효화**한다.

> ⚠️ 캐시 TTL(10분)을 기다리지 않고 **즉시 무효화**하는 이유: "껐는데 계속 온다"는 신뢰를 직접 훼손하기 때문이다.

---

## 6. 단말 관리 API

### 6.1 `POST /v1/users/{user_id}/devices` — 단말 등록

```json
{
  "device_token": "fcm-dGhpcyBpcyBhIHNhbXBsZSB0b2tlbg",
  "platform": "ANDROID",
  "app_version": "5.2.1"
}
```

| 응답 | 상황 |
| :---: | --- |
| `201 Created` | 신규 등록 |
| `200 OK` | 이미 등록된 토큰 → `last_logged_in_at` 갱신 |
| `409 TOKEN_OWNED_BY_OTHER_USER` | 같은 토큰이 다른 사용자에 귀속 → **기존 귀속을 해제하고 재할당** 후 `200` |

> 단말 토큰은 기기 초기화·앱 재설치로 **재발급·재할당**된다. 유니크 제약과 재할당 규칙이 없으면 **오배송**이 발생한다.

### 6.2 `DELETE /v1/users/{user_id}/devices/{device_token}`

로그아웃·앱 삭제 시 호출. `is_active = FALSE`로 **소프트 삭제**하고 `device:{user_id}` 캐시를 무효화한다. → `204 No Content`

---

## 7. 템플릿 API

### 7.1 `POST /v1/templates` — 템플릿 등록

```json
{
  "id": "ITEM_RESTOCK",
  "channel": "PUSH",
  "locale": "ko-KR",
  "category": "MARKETING",
  "title": "지금 [item_name]을 주문 또는 예약하세요!",
  "body": "여러분이 꿈꿔온 그 상품을 우리가 준비했습니다. [item_name]이 다시 입고되었습니다! [date]까지만 주문 가능합니다!",
  "content_type": "text/plain",
  "required_params": ["item_name", "date"]
}
```

**응답 `201 Created`**

```json
{ "id": "ITEM_RESTOCK", "version": 4, "channel": "PUSH", "locale": "ko-KR", "is_active": true }
```

> 같은 `id`로 다시 등록하면 **`version`이 증가**한다. 기존 버전은 유지되어 과거 알림의 재현이 가능하다.

### 7.2 `GET /v1/templates/{template_id}`

`?version=3` 생략 시 최신 활성 버전을 반환한다.

---

## 8. 인바운드 웹훅

### 8.1 `POST /v1/webhooks/delivery` — 제3자 전달 결과 수신

Sendgrid·Twilio 등이 **전달 완료·반송·열람·클릭**을 회신하는 엔드포인트다.

```json
{
  "provider": "SENDGRID",
  "events": [
    { "notification_id": "c74d1e88-...", "event": "delivered", "timestamp": 1786435807 },
    { "notification_id": "c74d1e88-...", "event": "open",      "timestamp": 1786435900 },
    { "notification_id": "9b2f0a11-...", "event": "bounce",    "timestamp": 1786435810,
      "reason": "550 5.1.1 user unknown" }
  ]
}
```

| 제3자 이벤트 | 매핑되는 알림 상태 | 후속 처리 |
| --- | --- | --- |
| `delivered` | `DELIVERED` | — |
| `open` | `OPENED` | 확인율 집계 |
| `click` | `CLICKED` | 클릭률 집계 |
| `bounce` (영구) | `FAILED_PERMANENT` | ⚠️ 이메일 주소 무효 표시 |
| `unsubscribe` | — | ⚠️ **`notification_setting.opt_in = false`로 자동 반영** |
| APNS `410 Gone` | `FAILED_PERMANENT` | ⚠️ `device.is_active = FALSE` |

> 웹훅도 **서명 검증**을 거친다. 검증되지 않은 요청은 `401`. 처리는 **멱등**해야 한다(같은 이벤트가 재전송될 수 있음).

---

## 9. `GET /v1/health`

```json
{
  "status": "degraded",
  "components": {
    "notification_server": "healthy",
    "metadata_db":         "healthy",
    "cache":               "healthy",
    "log_store":           "healthy",
    "queue":               "healthy",
    "providers": {
      "APNS":     { "status": "healthy",  "circuit": "closed" },
      "FCM":      { "status": "healthy",  "circuit": "closed" },
      "TWILIO":   { "status": "degraded", "circuit": "open", "fallback": "NEXMO" },
      "SENDGRID": { "status": "healthy",  "circuit": "closed" }
    }
  },
  "queue_depth": { "push.ios": 120, "push.android": 95, "sms": 18400, "email": 310 }
}
```

> ⚠️ `sms` 큐 깊이 18,400은 **Twilio 서킷 오픈으로 적체된 상태**를 보여준다.
> 이 값이 [06 운영 설계](06-notification-operations.md)의 **핵심 모니터링 지표**다.

---

## 10. API ↔ 요구사항 매핑

| API | 충족 요구사항 |
| --- | --- |
| `POST /v1/notifications` | FR-1 다채널, FR-2 팬아웃, FR-3 opt-out, FR-4 템플릿, FR-7 전송률 제한 |
| `event_id` 멱등 규약 | NFR-2 중복 최소화 |
| appKey/appSecret 서명 | NFR-6 보안 |
| `202 Accepted` 접수 응답 | NFR-3 연성 실시간 |
| `GET /v1/notifications/{id}` | FR-5 상태 조회 |
| 수신 설정 API | FR-3 opt-out |
| 단말 관리 API | FR-2 팬아웃 |
| 템플릿 API | FR-4 템플릿 |
| 웹훅 | FR-8 이벤트 추적 |
| `503 LOG_STORE_UNAVAILABLE` | NFR-1 무손실 (fail-close) |

---

➡️ 다음: [05 안정성 설계](05-notification-reliability.md)
