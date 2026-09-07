# 당타 MVP 화면 목업

`docs/mvp/` 범위에 맞춘 13개 화면 하이파이 목업. 탭 4개(홈·커뮤니티·채팅·마이).

## 발행된 캔버스

https://claude.ai/code/artifact/de79815a-90b8-4c4b-b9df-61389f2ed3e3

(소유자가 공유 메뉴에서 링크를 열어야 팀원이 볼 수 있음)

## 전체 기획 목업과 차이

| 뺀 것 | 이유 |
|---|---|
| 중고책 탭, 소모임 탭, 시간표 | MVP 범위 밖. '책'은 중고거래 카테고리로 |
| 채팅 약속잡기 카드, 이미지 전송 | 폴링 채팅 = 텍스트만 |
| 매너온도 부정 평가 / 재거래희망률 | 좋았던 점 태그만, 태그당 +0.5℃ |
| 커뮤니티 핫게시판, 투표, 대댓글, 스크랩 | 배치·부가기능 제외 |
| 알림함, 상품 가격제안·거래방식 | 범위 축소 |

## 화면 목록

| 순서 | 화면 | 파일 | 라우트 |
|---|---|---|---|
| 1 | 로그인 | Login.dc.html | `/login` |
| 2 | 가입 1 · 이메일 인증 | SignupEmail.dc.html | `/signup` |
| 3 | 가입 2 · 닉네임·비밀번호 | SignupProfile.dc.html | `/signup` |
| 4 | 홈 · 중고거래 피드 | Main.dc.html | `/` |
| 5 | 상품 상세 | ProductDetail.dc.html | `/products/:id` |
| 6 | 상품 등록 | ProductNew.dc.html | `/products/new` |
| 7 | 커뮤니티 피드 | Community.dc.html | `/community` |
| 8 | 게시글 상세 · 댓글 | PostDetail.dc.html | `/community/posts/:id` |
| 9 | 글쓰기 | PostNew.dc.html | `/community/write` |
| 10 | 채팅 목록 | ChatList.dc.html | `/chat` |
| 11 | 채팅방 (폴링) | ChatRoom.dc.html | `/chat/:roomId` |
| 12 | 마이 · 매너온도 | MyPage.dc.html | `/my` |
| 13 | 거래 후 매너 평가 | MannerReview.dc.html | `/manner/review` |

## 수정

`*.dc.html` 편집 후 `design` 스킬 `seed-canvas.mjs`로 재시드 → 같은 URL 재발행.
