# MVP API 명세

- Base URL: `http://localhost:8080` (개발) → 배포 시 실제 도메인
- Prefix: `/api`
- 포맷: JSON, 필드 `camelCase`
- 인증: `Authorization: Bearer <accessToken>` (로그인 시 발급, 유효기간 7일, 리프레시 없음)
- 응답: 성공은 `{ "data": ... }`, 실패는 `{ "error": { "code", "message" } }`
- 페이지네이션: `?page=0&size=20` (0부터), 응답에 `page` 메타
- 권한 없으면 `401`, 남의 리소스 수정 시 `403`, 없으면 `404`

```jsonc
// 목록 응답 공통 형태
{
  "data": [ /* ... */ ],
  "page": { "number": 0, "size": 20, "totalElements": 42, "totalPages": 3 }
}
```

---

## 1. 인증 / 사용자

| Method | Path | 설명 | 인증 |
|---|---|---|---|
| POST | `/api/auth/email/send-code` | 인증코드 발송 (개발모드: 코드 응답에 포함) | X |
| POST | `/api/auth/signup` | 회원가입 | X |
| POST | `/api/auth/login` | 로그인 → 토큰 발급 | X |
| GET | `/api/users/me` | 내 정보 | O |
| PATCH | `/api/users/me` | 내 프로필 수정 | O |
| GET | `/api/users/{id}` | 다른 사용자 공개 프로필 | O |

```jsonc
// POST /api/auth/email/send-code
{ "email": "minseo@hankuk.ac.kr" }
// 200
{ "data": { "universityName": "한국대학교", "devCode": "481920", "expiresInSec": 600 } }
// devCode는 개발 모드에서만. 실서비스면 제거.
// 404 { "error": { "code": "UNIVERSITY_NOT_FOUND", "message": "등록되지 않은 학교 이메일입니다." } }
```

```jsonc
// POST /api/auth/signup
{ "email": "minseo@hankuk.ac.kr", "code": "481920", "password": "pw12345!", "nickname": "공대냉장고요정", "department": "전자공학과" }
// 201
{ "data": { "token": "eyJ...", "user": { "id": 3, "nickname": "공대냉장고요정", "mannerTemperature": 36.5 } } }
// 409 { "error": { "code": "NICKNAME_TAKEN" } }
```

```jsonc
// POST /api/auth/login
{ "email": "minseo@hankuk.ac.kr", "password": "pw12345!" }
// 200
{ "data": { "token": "eyJ...", "user": { "id": 3, "nickname": "공대냉장고요정", "mannerTemperature": 42.3, "profileImageUrl": null } } }
```

```jsonc
// GET /api/users/me  → 200
{ "data": {
  "id": 3, "email": "minseo@hankuk.ac.kr", "nickname": "공대냉장고요정",
  "department": "전자공학과", "profileImageUrl": null,
  "mannerTemperature": 42.3, "universityName": "한국대학교",
  "unreadChatCount": 2
}}

// PATCH /api/users/me
{ "nickname": "새닉네임", "department": "컴퓨터공학과", "profileImageUrl": "/uploads/ab.jpg" }

// GET /api/users/5 → 200 (공개 프로필)
{ "data": { "id": 5, "nickname": "서연", "department": "경영학과",
  "mannerTemperature": 38.0, "sellingCount": 3 } }
```

---

## 2. 중고거래

| Method | Path | 설명 | 인증 |
|---|---|---|---|
| GET | `/api/categories` | 카테고리 목록 | O |
| GET | `/api/products` | 상품 목록/검색 | O |
| POST | `/api/products` | 상품 등록 | O |
| GET | `/api/products/{id}` | 상품 상세 (조회수 +1) | O |
| PATCH | `/api/products/{id}` | 수정 | 작성자 |
| DELETE | `/api/products/{id}` | 삭제 | 작성자 |
| PATCH | `/api/products/{id}/status` | 거래상태 변경 | 작성자 |
| POST | `/api/products/{id}/favorite` | 찜 | O |
| DELETE | `/api/products/{id}/favorite` | 찜 취소 | O |
| GET | `/api/users/me/products` | 내가 등록한 상품 | O |
| GET | `/api/users/me/favorites` | 내가 찜한 상품 | O |
| POST | `/api/uploads/image` | 이미지 업로드 (multipart) | O |

**GET `/api/products` 쿼리 파라미터**

| 파라미터 | 예 | 설명 |
|---|---|---|
| `keyword` | `아이패드` | 제목 LIKE 검색 |
| `categoryId` | `2` | 카테고리 필터 |
| `status` | `ON_SALE` | 상태 필터 (기본: 전체) |
| `sort` | `latest` \| `priceAsc` \| `priceDesc` | 정렬 (기본 latest) |
| `page`, `size` | `0`, `20` | 페이지 |

```jsonc
// GET /api/products?keyword=냉장고&categoryId=2&page=0&size=20  → 200
{
  "data": [
    {
      "id": 12, "title": "자취 미니 냉장고", "price": 60000, "status": "ON_SALE",
      "thumbnailUrl": "/uploads/fridge1.jpg", "campusPlace": "공대 5호관",
      "favoriteCount": 3, "chatCount": 2, "createdAt": "2026-09-08T14:31:00",
      "seller": { "id": 3, "nickname": "공대냉장고요정", "mannerTemperature": 42.3 }
    }
  ],
  "page": { "number": 0, "size": 20, "totalElements": 1, "totalPages": 1 }
}
```

```jsonc
// POST /api/products
{
  "title": "자취 미니 냉장고 (1년 사용)",
  "description": "소음 없어요. 공대 5호관 직거래.",
  "price": 60000,
  "categoryId": 2,
  "campusPlace": "공대 5호관",
  "imageUrls": ["/uploads/fridge1.jpg", "/uploads/fridge2.jpg"]
}
// 201 { "data": { "id": 12, "status": "ON_SALE" } }

// GET /api/products/12  → 200 (상세)
{ "data": {
  "id": 12, "title": "...", "description": "...", "price": 60000, "status": "ON_SALE",
  "campusPlace": "공대 5호관", "viewCount": 128, "favoriteCount": 3,
  "images": ["/uploads/fridge1.jpg", "/uploads/fridge2.jpg"],
  "isFavorite": false, "isMine": false,
  "seller": { "id": 3, "nickname": "공대냉장고요정", "mannerTemperature": 42.3, "profileImageUrl": null },
  "createdAt": "2026-09-08T14:31:00"
}}

// PATCH /api/products/12/status
{ "status": "RESERVED" }
// 200 { "data": { "id": 12, "status": "RESERVED" } }
// 409 { "error": { "code": "INVALID_STATUS_TRANSITION" } }
```

```jsonc
// POST /api/uploads/image   (Content-Type: multipart/form-data, field: "file")
// 200 { "data": { "url": "/uploads/8f3a2b.jpg" } }
// 서버는 파일을 로컬 uploads 폴더에 저장하고 정적 경로를 돌려준다.
```

---

## 3. 커뮤니티

| Method | Path | 설명 | 인증 |
|---|---|---|---|
| GET | `/api/boards` | 게시판 목록 | O |
| GET | `/api/boards/{slug}/posts` | 게시판 글 목록 | O |
| POST | `/api/posts` | 글 작성 | O |
| GET | `/api/posts/{id}` | 글 상세 (조회수 +1) | O |
| PATCH | `/api/posts/{id}` | 수정 | 작성자 |
| DELETE | `/api/posts/{id}` | 삭제 | 작성자 |
| POST | `/api/posts/{id}/like` | 좋아요 | O |
| DELETE | `/api/posts/{id}/like` | 좋아요 취소 | O |
| GET | `/api/posts/{id}/comments` | 댓글 목록 | O |
| POST | `/api/posts/{id}/comments` | 댓글 작성 | O |
| DELETE | `/api/comments/{id}` | 댓글 삭제 | 작성자 |
| GET | `/api/users/me/posts` | 내가 쓴 글 | O |

```jsonc
// GET /api/boards → 200
{ "data": [
  { "id": 1, "name": "자유", "slug": "free" },
  { "id": 2, "name": "정보", "slug": "info" },
  { "id": 3, "name": "질문", "slug": "qna" }
]}

// GET /api/boards/free/posts?sort=latest&page=0&size=20  → 200
{ "data": [
  { "id": 40, "title": "중간고사 언제부터 공부함?", "preview": "전공 3개인데...",
    "author": { "type": "ANONYMOUS", "label": "익명" },
    "likeCount": 42, "commentCount": 18, "createdAt": "2026-09-08T12:00:00" }
], "page": { "number": 0, "size": 20, "totalElements": 1, "totalPages": 1 } }
// sort: latest | popular

// POST /api/posts
{ "boardId": 1, "title": "중간고사 언제부터 공부함?", "content": "전공 3개인데...", "anonymous": true }
// 201 { "data": { "id": 40 } }

// GET /api/posts/40 → 200
{ "data": {
  "id": 40, "board": { "slug": "free", "name": "자유" },
  "title": "중간고사 언제부터 공부함?", "content": "...",
  "author": { "type": "ANONYMOUS", "label": "익명(글쓴이)" },
  "likeCount": 42, "commentCount": 18, "viewCount": 210,
  "isLiked": false, "isMine": false,
  "createdAt": "2026-09-08T12:00:00"
}}
// 실명 글이면 author: { "type": "USER", "id": 3, "label": "공대냉장고요정" }

// POST /api/posts/40/comments
{ "content": "2주 전부터요", "anonymous": true }
// 201 { "data": { "id": 501, "author": { "type": "ANONYMOUS", "label": "익명" } } }
```

---

## 4. 채팅 (폴링)

| Method | Path | 설명 | 인증 |
|---|---|---|---|
| GET | `/api/chat/rooms` | 내 채팅방 목록 | O |
| POST | `/api/chat/rooms` | 방 생성 또는 기존 방 반환 | O |
| GET | `/api/chat/rooms/{id}` | 방 정보 (상대·상품) | 참여자 |
| GET | `/api/chat/rooms/{id}/messages` | 메시지 조회 (폴링) | 참여자 |
| POST | `/api/chat/rooms/{id}/messages` | 메시지 전송 | 참여자 |
| POST | `/api/chat/rooms/{id}/read` | 읽음 처리 | 참여자 |
| POST | `/api/chat/rooms/{id}/complete` | 거래완료 (판매자만) | 판매자 |
| GET | `/api/chat/unread-count` | 전체 안읽음 수 | O |

```jsonc
// POST /api/chat/rooms
{ "productId": 12 }
// 200 (기존 방 있으면 그대로) { "data": { "id": 88 } }

// GET /api/chat/rooms → 200
{ "data": [
  { "id": 88, "product": { "id": 12, "title": "자취 미니 냉장고", "price": 60000, "thumbnailUrl": "/uploads/fridge1.jpg", "status": "ON_SALE" },
    "opponent": { "id": 5, "nickname": "서연", "mannerTemperature": 38.0 },
    "lastMessage": "네 6시에 봬요", "lastMessageAt": "2026-09-08T14:36:00",
    "unreadCount": 2 }
]}

// GET /api/chat/rooms/88/messages?afterId=140   (afterId 없으면 최근 50개)
// 200
{ "data": [
  { "id": 141, "senderId": 5, "content": "네 6시에 봬요", "createdAt": "2026-09-08T14:36:00", "mine": false }
]}
// 프론트: 3~5초마다 마지막 id를 afterId로 넘겨 새 메시지만 가져옴

// POST /api/chat/rooms/88/messages
{ "content": "좋아요! 그때 뵐게요" }
// 201 { "data": { "id": 142, "createdAt": "2026-09-08T14:37:00" } }

// POST /api/chat/rooms/88/read
{ "lastMessageId": 142 }
// 200 { "data": { "ok": true } }

// POST /api/chat/rooms/88/complete   (판매자가 호출)
// 200 { "data": { "productId": 12, "status": "SOLD", "reviewTargetId": 5 } }
// → 상품 SOLD, 매너평가 대상 확정
```

---

## 5. 매너온도

| Method | Path | 설명 | 인증 |
|---|---|---|---|
| GET | `/api/manner/pending` | 내가 평가해야 할 거래 목록 | O |
| POST | `/api/manner/reviews` | 매너 평가 작성 | O |
| GET | `/api/users/{id}/manner-reviews` | 받은 후기 (태그 통계) | O |

```jsonc
// GET /api/manner/pending → 200
{ "data": [
  { "productId": 12, "productTitle": "자취 미니 냉장고",
    "target": { "id": 5, "nickname": "서연" }, "tradedAt": "2026-09-10" }
]}

// POST /api/manner/reviews
{ "productId": 12, "targetId": 5, "tags": ["KIND", "ON_TIME"], "comment": "시간 잘 지키시고 친절해요!" }
// 201 { "data": { "scoreDelta": 1.0, "targetTemperature": 39.0 } }
// comment 는 선택
// 409 { "error": { "code": "ALREADY_REVIEWED" } }

// GET /api/users/5/manner-reviews → 200
{ "data": {
  "totalCount": 17,
  "tagCounts": { "KIND": 12, "ON_TIME": 9, "FAST_REPLY": 5, "GOOD_ITEM": 7, "GENEROUS": 2 },
  "comments": [
    { "text": "약속 시간에 딱 맞춰 오셨고 물건도 설명 그대로였어요!", "createdAt": "2026-09-10" },
    { "text": "채팅 답장이 엄청 빨라서 편하게 거래했습니다.", "createdAt": "2026-09-05" }
  ]
}}
```

태그 코드: `KIND` 친절 / `ON_TIME` 시간약속 / `GOOD_ITEM` 상품좋음 / `FAST_REPLY` 응답빠름 / `GENEROUS` 좋은나눔

---

## 6. 신고

| Method | Path | 설명 | 인증 |
|---|---|---|---|
| POST | `/api/reports` | 신고 (값만 저장) | O |

```jsonc
// POST /api/reports
{ "targetType": "PRODUCT", "targetId": 12, "reason": "SCAM", "detail": "사기 의심" }
// 201 { "data": { "id": 7 } }
```
`targetType`: PRODUCT / POST / COMMENT / USER · `reason`: SCAM / ABUSE / SPAM / ETC

---

## 7. 에러 코드

| HTTP | code | 상황 |
|---|---|---|
| 400 | `VALIDATION_FAILED` | 입력값 오류 |
| 401 | `UNAUTHENTICATED` | 토큰 없음/만료 |
| 403 | `FORBIDDEN` | 남의 리소스 |
| 404 | `NOT_FOUND` | 리소스 없음 |
| 409 | `NICKNAME_TAKEN` / `EMAIL_TAKEN` / `ALREADY_REVIEWED` / `INVALID_STATUS_TRANSITION` | 충돌 |
| 400 | `INVALID_CODE` / `CODE_EXPIRED` | 이메일 인증 실패 |

---

## 8. 구현 순서 (백엔드)

1. 프로젝트 셋업 + 공통 (`@RestControllerAdvice` 예외 처리, 응답 래퍼, CORS, Security 필터, JWT 유틸)
2. 인증 (`send-code` → `signup` → `login` → `/users/me`)
3. 카테고리·상품 CRUD → 목록/검색 → 상태변경 → 찜 → 이미지 업로드
4. 게시판·글·댓글·좋아요
5. 채팅 (방 생성 → 메시지 조회/전송 → 읽음 → complete)
6. 매너온도 (`pending` → `reviews`, `MannerScorePolicy`로 계산 분리)
7. 신고, 안읽음 카운트, 시연용 시드 보강
