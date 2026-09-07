# 04. API 명세 — 당타(DangTa)

- Base URL: `https://api.dangta.app`
- API prefix: `/api/v1`
- 포맷: JSON (UTF-8). 요청/응답 필드는 `camelCase`.
- 인증: `Authorization: Bearer <accessToken>` (Refresh는 httpOnly 쿠키)
- 관리자 API: `/api/v1/admin/**`, `role=ADMIN` 필요

---

## 1. 공통 규약

### 1.1 성공 응답

단건:
```json
{ "data": { "id": 12, "title": "..." } }
```

목록(커서 페이지네이션):
```json
{
  "data": [ { "id": 42 }, { "id": 41 } ],
  "page": { "nextCursor": "eyJidW1wZWRBdCI6..." , "hasNext": true }
}
```

### 1.2 에러 응답

```json
{
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "상품을 찾을 수 없습니다.",
    "fieldErrors": [ { "field": "price", "message": "0 이상이어야 합니다." } ]
  }
}
```

| HTTP | 대표 code | 상황 |
|---|---|---|
| 400 | `VALIDATION_FAILED` | 입력 검증 실패 |
| 401 | `UNAUTHENTICATED`, `TOKEN_EXPIRED` | 미인증/만료 |
| 403 | `FORBIDDEN`, `NOT_VERIFIED`, `SANCTIONED` | 권한 없음/미인증/제재 |
| 404 | `*_NOT_FOUND` | 리소스 없음 |
| 409 | `NICKNAME_TAKEN`, `ALREADY_EXISTS`, `INVALID_STATE` | 충돌/상태 위반 |
| 429 | `RATE_LIMITED` | 요청 과다 |
| 500 | `INTERNAL_ERROR` | 서버 오류 |

### 1.3 공통 쿼리 파라미터 (목록)

| 파라미터 | 설명 |
|---|---|
| `cursor` | 이전 응답의 `nextCursor` |
| `size` | 페이지 크기 (기본 20, 최대 50) |
| `sort` | 도메인별 허용값 (예: `latest`, `popular`, `price_asc`) |
| `q` | 검색어 |

### 1.4 이미지 업로드 흐름

```
POST /api/v1/uploads/presign  { "domain": "product", "contentType": "image/jpeg", "count": 3 }
 → 200 { "data": [ { "uploadUrl": "https://s3...", "key": "product/2026/uuid.jpg" } ] }
클라이언트가 uploadUrl 로 PUT 업로드
생성 API 호출 시 body 에 "imageKeys": ["product/2026/uuid.jpg", ...] 전달
```

### 1.5 인증/제재 게이트

- `user.status != ACTIVE` → 쓰기 API 전부 `403 NOT_VERIFIED` 또는 `SANCTIONED`
- `WRITE_BAN` 제재 중 → 글/댓글/상품/채팅 생성 `403 SANCTIONED` (읽기는 허용)
- 차단 관계 → 상대 리소스 조회 시 `404`, 채팅/거래 시도 시 `403 BLOCKED`

---

## 2. 인증 · 대학

| Method | Path | 설명 | 인증 |
|---|---|---|---|
| POST | `/auth/email/request-code` | 이메일로 인증코드 발송 | 불필요 |
| POST | `/auth/email/verify` | 코드 검증 → 토큰 발급 (+ 신규면 온보딩 필요 플래그) | 불필요 |
| POST | `/auth/token/refresh` | Refresh 쿠키로 Access 재발급 | 쿠키 |
| POST | `/auth/logout` | 현재 세션 로그아웃 | 필요 |
| POST | `/auth/logout-all` | 전체 세션 로그아웃 | 필요 |
| POST | `/auth/reverify/request-code` | 재인증 코드 발송 | 필요 |
| POST | `/auth/reverify/verify` | 재인증 완료 | 필요 |
| DELETE | `/auth/account` | 회원 탈퇴 | 필요 |
| GET | `/universities/resolve?email=` | 이메일 도메인으로 대학 조회 | 불필요 |
| GET | `/universities/{id}/campus-locations` | 캠퍼스 위치 목록 | 필요 |

요청/응답 예:

```
POST /auth/email/request-code
{ "email": "minseo@snu.ac.kr", "purpose": "SIGNUP" }
→ 200 { "data": { "university": { "id": 1, "name": "서울대학교" }, "resendAfterSec": 60, "expiresInSec": 300 } }
→ 404 { "error": { "code": "UNIVERSITY_NOT_FOUND", "message": "등록되지 않은 학교 이메일입니다." } }
```

```
POST /auth/email/verify
{ "email": "minseo@snu.ac.kr", "code": "481920", "purpose": "SIGNUP" }
→ 200
{
  "data": {
    "accessToken": "eyJ...",
    "accessTokenExpiresIn": 900,
    "needsOnboarding": true,
    "user": { "id": 55, "status": "PENDING_ONBOARDING" }
  }
}
```

---

## 3. 사용자 · 프로필 · 매너온도

| Method | Path | 설명 |
|---|---|---|
| GET | `/me` | 내 정보 (user + profile + mannerScore + 미읽음 카운트) |
| PATCH | `/me/onboarding` | 온보딩 완료 (nickname, imageKey?, admissionYear, department?) |
| PATCH | `/me/profile` | 프로필 수정 (imageKey?, department?, bio?) |
| PATCH | `/me/nickname` | 닉네임 변경 (제한 정책 적용) |
| GET | `/me/notification-setting` | 알림 설정 조회 |
| PATCH | `/me/notification-setting` | 알림 설정 수정 |
| GET | `/users/{id}` | 타인 프로필 (매너온도, 재거래희망률, 판매중 목록 요약) |
| GET | `/users/{id}/products?status=ON_SALE` | 특정 사용자 판매 목록 |
| GET | `/users/{id}/manner-reviews` | 받은 후기 목록 |
| GET | `/nicknames/check?value=` | 닉네임 중복 확인 |

| Method | Path | 설명 |
|---|---|---|
| GET | `/manner/pending` | 내가 작성해야 할 매너 평가 목록 |
| POST | `/manner/reviews` | 매너 평가 작성 |

```
POST /manner/reviews
{
  "chatRoomId": 88,
  "revieweeId": 42,
  "tradeType": "PRODUCT",
  "tradeRefId": 301,
  "positiveTags": ["KIND", "ON_TIME"],
  "negativeTags": [],
  "wantRebuy": true,
  "comment": "친절하고 시간 잘 지키세요!"
}
→ 201 { "data": { "id": 900, "appliedDelta": 0.4, "revieweeTemperature": 37.2 } }
→ 409 { "error": { "code": "ALREADY_REVIEWED" } }
```

---

## 4. 중고거래

| Method | Path | 설명 | 인증 |
|---|---|---|---|
| GET | `/products` | 피드/검색. 파라미터: `sort(latest\|popular\|price_asc\|price_desc)`, `categorySlug`, `minPrice`, `maxPrice`, `campusLocationId`, `onSaleOnly`, `q`, `cursor`, `size` | 필요 |
| POST | `/products` | 상품 등록 | 필요 |
| GET | `/products/{id}` | 상세 (조회수 증가) | 필요 |
| PATCH | `/products/{id}` | 수정 | 본인 |
| DELETE | `/products/{id}` | 삭제(soft) | 본인 |
| POST | `/products/{id}/bump` | 끌올 | 본인 |
| PATCH | `/products/{id}/status` | 상태 변경 `{ status, reservedBuyerId? }` | 본인 |
| POST | `/products/{id}/favorite` | 찜 | 필요 |
| DELETE | `/products/{id}/favorite` | 찜 취소 | 필요 |
| GET | `/me/favorites?targetType=PRODUCT` | 내 찜 목록 | 필요 |
| GET | `/me/products?role=selling\|bought` | 내 판매/구매 내역 | 필요 |
| GET | `/product-categories` | 카테고리 목록 | 필요 |

```
POST /products
{
  "title": "자취 냉장고 (1년 사용)",
  "content": "이사 가서 팝니다. 상태 좋아요.",
  "categorySlug": "home-appliance",
  "price": 60000,
  "tradeMethod": "MEET",
  "campusLocationId": 12,
  "imageKeys": ["product/2026/a.jpg", "product/2026/b.jpg"]
}
→ 201 { "data": { "id": 301, "status": "ON_SALE" } }
```

```
PATCH /products/301/status
{ "status": "RESERVED", "reservedBuyerId": 42 }
→ 200 { "data": { "id": 301, "status": "RESERVED" } }
→ 409 { "error": { "code": "INVALID_STATE", "message": "이미 거래완료된 상품입니다." } }
```

---

## 5. 중고책

| Method | Path | 설명 |
|---|---|---|
| GET | `/books/search?q=&isbn=` | 도서 마스터 검색 (외부 API + 내부) |
| POST | `/books` | 신규 도서 마스터 등록 (검수 큐 가능) |
| GET | `/books/{id}` | 도서 상세 + 이 학교 매물 수 |
| GET | `/books/{id}/listings` | 특정 도서의 매물 목록 |
| GET | `/book-listings` | 매물 피드. 파라미터: `sort(latest\|discount\|price_asc)`, `department`, `grade`, `courseId`, `condition`, `q` |
| POST | `/book-listings` | 매물 등록 `{ bookId?, newBook?, price, condition, hasNotes, content?, courseIds[], campusLocationId?, imageKeys[] }` |
| GET | `/book-listings/{id}` | 매물 상세 |
| PATCH | `/book-listings/{id}` | 수정 |
| DELETE | `/book-listings/{id}` | 삭제 |
| POST | `/book-listings/{id}/bump` | 끌올 |
| PATCH | `/book-listings/{id}/status` | 상태 변경 |
| POST/DELETE | `/book-listings/{id}/favorite` | 찜/취소 |
| GET | `/courses/{id}/book-listings` | 강의로 교재 찾기 |

```
POST /book-listings
{
  "newBook": { "isbn13": "9791162241974", "title": "일반물리학", "author": "Halliday", "listPrice": 38000 },
  "price": 15000,
  "condition": "MID",
  "hasNotes": true,
  "courseIds": [4412],
  "campusLocationId": 3,
  "imageKeys": ["book/2026/x.jpg"]
}
→ 201 { "data": { "id": 720, "discountRate": 0.61 } }
```

---

## 6. 소모임

| Method | Path | 설명 |
|---|---|---|
| GET | `/groups` | 목록. `category`, `sort(active\|new\|members)`, `q` |
| POST | `/groups` | 개설 (개설자 자동 OWNER) |
| GET | `/groups/{id}` | 상세 (비멤버는 소개까지) |
| PATCH | `/groups/{id}` | 수정 (OWNER/MANAGER) |
| DELETE | `/groups/{id}` | 해산 (OWNER) |
| POST | `/groups/{id}/join` | 가입/신청 `{ answer? }` |
| DELETE | `/groups/{id}/leave` | 탈퇴 |
| GET | `/groups/{id}/members?status=` | 멤버/신청 목록 |
| PATCH | `/groups/{id}/members/{userId}` | 승인/거절/역할변경/강퇴 `{ action: APPROVE\|REJECT\|SET_MANAGER\|SET_MEMBER\|KICK }` |
| GET | `/groups/{id}/posts` | 모임 게시판 |
| POST | `/groups/{id}/posts` | 글 작성 `{ title, content, isNotice? }` |
| GET/POST | `/groups/{id}/posts/{postId}/comments` | 댓글 |
| GET | `/groups/{id}/schedules` | 일정 목록 |
| POST | `/groups/{id}/schedules` | 일정 생성 |
| PUT | `/groups/{id}/schedules/{sid}/rsvp` | 참석 응답 `{ status }` |
| GET | `/me/groups` | 내 모임 (승인됨) + 신청중 |
| GET | `/group-categories` | 카테고리 목록 |

```
PATCH /groups/15/members/42
{ "action": "APPROVE" }
→ 200 { "data": { "userId": 42, "status": "APPROVED" } }
```

---

## 7. 커뮤니티

| Method | Path | 설명 |
|---|---|---|
| GET | `/boards` | 우리 학교 게시판 목록 |
| GET | `/boards/{slug}/posts` | 게시글 목록 `sort(latest\|popular)`, `q` |
| GET | `/posts/hot` | 핫게시판 (최근 24h) |
| POST | `/posts` | 글 작성 `{ boardSlug, title, content, isAnonymous, imageKeys[], poll? }` |
| GET | `/posts/{id}` | 상세 (조회수 증가, 익명번호 포함) |
| PATCH | `/posts/{id}` | 수정 (본문/이미지) |
| DELETE | `/posts/{id}` | 삭제 |
| POST/DELETE | `/posts/{id}/like` | 좋아요/취소 |
| POST/DELETE | `/posts/{id}/scrap` | 스크랩/취소 |
| GET | `/me/scraps` | 스크랩 목록 |
| GET | `/me/posts` | 내가 쓴 글 |
| GET | `/posts/{id}/comments` | 댓글 트리 |
| POST | `/posts/{id}/comments` | 댓글/대댓글 `{ content, parentId?, isAnonymous }` |
| POST/DELETE | `/comments/{id}/like` | 댓글 좋아요 |
| DELETE | `/comments/{id}` | 댓글 삭제 |
| POST | `/posts/{id}/poll/vote` | 투표 `{ optionIds[] }` |

```
GET /posts/1024
→ 200
{
  "data": {
    "id": 1024,
    "board": { "slug": "free", "name": "자유게시판" },
    "title": "중간고사 언제부터 공부함?",
    "content": "...",
    "author": { "type": "ANONYMOUS", "anonNo": 0, "label": "익명(글쓴이)" },
    "isAnonymous": true,
    "likeCount": 42, "commentCount": 12, "scrapCount": 3,
    "liked": false, "scrapped": false,
    "images": [],
    "poll": null,
    "createdAt": "2026-09-06T12:00:00+09:00"
  }
}
```

```
POST /posts/1024/comments
{ "content": "2주 전부터요", "isAnonymous": true }
→ 201 { "data": { "id": 5001, "author": { "type": "ANONYMOUS", "anonNo": 3, "label": "익명3" } } }
```

---

## 8. 채팅

### 8.1 REST

| Method | Path | 설명 |
|---|---|---|
| GET | `/chat/rooms` | 내 채팅방 목록 (최근순, 미읽음 수) |
| POST | `/chat/rooms` | 방 생성/조회 `{ tradeType, tradeRefId }` → 기존 방 있으면 반환 |
| GET | `/chat/rooms/{id}` | 방 상세 (상대, 연결된 거래글 요약) |
| GET | `/chat/rooms/{id}/messages?cursor=` | 메시지 히스토리 (역순 페이징) |
| POST | `/chat/rooms/{id}/messages` | 메시지 전송(REST 폴백; 기본은 STOMP) |
| POST | `/chat/rooms/{id}/read` | 읽음 처리 `{ lastReadMessageId }` |
| POST | `/chat/rooms/{id}/appointment` | 약속 잡기 `{ at, campusLocationId, memo? }` |
| POST | `/chat/rooms/{id}/complete` | 거래완료 (판매자만) → 상품 SOLD + 평가요청 |
| POST | `/chat/rooms/{id}/leave` | 나가기 |
| GET | `/chat/unread-count` | 전체 미읽음 수 |

### 8.2 WebSocket (STOMP over SockJS)

```
연결:   wss://api.dangta.app/ws         (CONNECT 헤더에 Authorization: Bearer)
구독:   SUBSCRIBE /user/queue/errors
        SUBSCRIBE /topic/rooms/{roomId}
발행:   SEND /app/rooms/{roomId}/send      { "type": "TEXT", "content": "안녕하세요" }
        SEND /app/rooms/{roomId}/read      { "lastReadMessageId": 4501 }
        SEND /app/rooms/{roomId}/typing    { "typing": true }
```

수신 이벤트 (`/topic/rooms/{roomId}`):
```json
{ "event": "MESSAGE", "data": { "id": 4502, "senderId": 42, "type": "TEXT", "content": "네 안녕하세요", "createdAt": "..." } }
{ "event": "READ", "data": { "userId": 42, "lastReadMessageId": 4502 } }
{ "event": "SYSTEM", "data": { "id": 4503, "type": "SYSTEM", "content": "거래가 완료되었습니다." } }
```

- 재연결: 클라이언트 지수 백오프, 재연결 후 `GET /chat/rooms/{id}/messages?cursor=` 로 놓친 메시지 동기화
- 미접속(구독 없음) 사용자 → 웹푸시 발송

약속 payload 예:
```json
{ "type": "APPOINTMENT", "payload": { "at": "2026-09-10T15:00:00+09:00", "campusLocationId": 12, "campusLocationName": "중앙도서관 앞", "memo": "정문 계단" } }
```

---

## 9. 시간표

| Method | Path | 설명 |
|---|---|---|
| GET | `/semesters?current=true` | 학기 목록 |
| GET | `/courses?semesterId=&q=&professor=&courseNo=` | 강의 마스터 검색 |
| GET | `/courses/{id}` | 강의 상세 (시간, 관련 교재 수) |
| GET | `/me/timetables?semesterId=` | 내 시간표 목록 |
| POST | `/timetables` | 시간표 생성 `{ semesterId, name }` |
| PATCH | `/timetables/{id}` | 수정 `{ name?, isPrimary? }` |
| DELETE | `/timetables/{id}` | 삭제 |
| POST | `/timetables/{id}/courses` | 강의 추가 `{ courseId }` 또는 `{ customTitle, customTimes[], color }` |
| PATCH | `/timetables/{id}/courses/{tcId}` | 색상/메모 수정 |
| DELETE | `/timetables/{id}/courses/{tcId}` | 강의 제거 |
| GET | `/timetables/{id}/share` | 공유용 읽기 전용 스냅샷/이미지 URL |
| GET | `/share/timetables/{token}` | 공유 링크 조회 (비로그인 가능, 읽기전용) |

```
POST /timetables/33/courses
{ "courseId": 4412 }
→ 201 { "data": { "id": 8801, "conflicts": [] } }
→ 200 { "data": { "id": 8802, "conflicts": [ { "courseId": 4001, "title": "미적분학" } ] } }  // 경고, 추가는 됨
```

---

## 10. 알림

| Method | Path | 설명 |
|---|---|---|
| GET | `/notifications?cursor=&category=` | 알림함 |
| POST | `/notifications/read` | 읽음 처리 `{ ids?[] , all?: true }` |
| GET | `/notifications/unread-count` | 미읽음 수 |
| POST | `/push/subscriptions` | 웹푸시 구독 등록 `{ endpoint, keys: { p256dh, auth } }` |
| DELETE | `/push/subscriptions` | 구독 해제 `{ endpoint }` |

---

## 11. 신고 · 차단

| Method | Path | 설명 |
|---|---|---|
| POST | `/reports` | 신고 `{ targetType, targetId, reason, detail? }` |
| POST | `/blocks` | 차단 `{ userId }` |
| DELETE | `/blocks/{userId}` | 차단 해제 |
| GET | `/me/blocks` | 차단 목록 |

---

## 12. 검색

| Method | Path | 설명 |
|---|---|---|
| GET | `/search?q=&type=all\|product\|book\|group\|post` | 통합 검색 (type=all 은 도메인별 상위 N개) |
| GET | `/search/suggestions?q=` | 자동완성 (선택) |
| GET | `/search/popular` | 학교별 인기 검색어 (선택) |

```
GET /search?q=에어팟&type=all
→ 200
{
  "data": {
    "products": { "items": [ ... ], "total": 12 },
    "books": { "items": [], "total": 0 },
    "groups": { "items": [], "total": 0 },
    "posts": { "items": [ ... ], "total": 3 }
  }
}
```

---

## 13. 관리자 API (요약)

| Method | Path | 설명 |
|---|---|---|
| GET/POST/PATCH | `/admin/universities` | 대학 + 이메일 도메인 관리 |
| GET/POST/PATCH | `/admin/campus-locations` | 캠퍼스 위치 관리 |
| GET | `/admin/reports?status=PENDING` | 신고 큐 |
| POST | `/admin/reports/{id}/action` | 처리 `{ action: BLIND\|DELETE\|WARN\|SUSPEND\|DISMISS, memo, durationDays? }` |
| GET/PATCH | `/admin/users/{id}` | 사용자 상세/제재 |
| GET/POST/PATCH | `/admin/boards` | 게시판 CRUD |
| GET/POST/PATCH | `/admin/product-categories` / `/admin/group-categories` | 카테고리 CRUD |
| POST | `/admin/courses/import` | 강의 마스터 CSV 업로드 |
| POST | `/admin/books/{id}/approve` | 도서 마스터 검수 |
| GET | `/admin/metrics/overview` | 기본 지표 |

---

## 14. 상태코드 & 권한 매트릭스 (요약)

| 액션 | 비로그인 | PENDING_ONBOARDING | ACTIVE | WRITE_BAN | 작성자 | ADMIN |
|---|---|---|---|---|---|---|
| 콘텐츠 조회 | ❌ | ⚠️(온보딩 유도) | ✅ | ✅ | ✅ | ✅ |
| 상품/글/댓글 작성 | ❌ | ❌ | ✅ | ❌ | — | ✅ |
| 본인 콘텐츠 수정/삭제 | ❌ | ❌ | ✅ | ✅(삭제만) | ✅ | ✅ |
| 채팅 | ❌ | ❌ | ✅ | ❌ | — | ✅ |
| 신고/차단 | ❌ | ✅ | ✅ | ✅ | — | ✅ |
| 관리자 API | ❌ | ❌ | ❌ | ❌ | — | ✅ |
