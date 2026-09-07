# 당타 화면 목업

두 종류가 있음:

| 폴더 | 내용 | 캔버스 |
|---|---|---|
| `wireframes/` (루트) | **전체 기획** 18개 화면. 5탭. 발표·로드맵용 | https://claude.ai/code/artifact/7a4d3a85-0cac-43f0-9b7a-d5acbb1da27c |
| [`wireframes/mvp/`](./mvp/) | **학기 구현 범위(MVP)** 13개 화면. 4탭. 실제로 이걸 만듦 | https://claude.ai/code/artifact/de79815a-90b8-4c4b-b9df-61389f2ed3e3 |

둘 다 모바일(390px) 하이파이. Noto Sans KR · 틸(#0EA5A4) + 화이트. `docs/06-design-system.md` 적용.
사진 자리는 카테고리별 톤 블록으로 표현.

---

## 전체 기획 목업 (아래는 `wireframes/` 루트 기준)

## 발행된 캔버스

https://claude.ai/code/artifact/7a4d3a85-0cac-43f0-9b7a-d5acbb1da27c

공유: 소유자가 공유 메뉴에서 링크를 열어야 다른 사람이 볼 수 있음(공개 링크 가능).
편집 권한이 있으면 Claude Design 캔버스 에디터로 요소 직접 수정 가능.

## 파일

- `*.dc.html` — 화면별 아트보드 소스 (Design Components 포맷)
- `canvas.json` — 캔버스 배치 · 섹션 메모
- `dangta-wireframes.html` — 시드 결과물(배포본). 직접 편집하지 말 것

## 수정 방법

`*.dc.html` 편집 후 `design` 스킬의 `seed-canvas.mjs`로 재시드 → 같은 Artifact URL로 재발행.

## 화면 목록

| 순서 | 화면 | 파일 |
|---|---|---|
| 1 | 로그인 / 시작 | Login.dc.html |
| 2 | 학교 이메일 인증 | EmailVerify.dc.html |
| 3 | 온보딩 프로필 | Onboarding.dc.html |
| 4 | 홈 · 중고거래 피드 | Main.dc.html |
| 5 | 상품 상세 | ProductDetail.dc.html |
| 6 | 상품 글쓰기 | ProductNew.dc.html |
| 7 | 채팅 목록 | ChatList.dc.html |
| 8 | 채팅방 · 약속잡기 | ChatRoom.dc.html |
| 9 | 거래 후 매너 평가 | MannerReview.dc.html |
| 10 | 중고책 피드 | Books.dc.html |
| 11 | 중고책 팔기 (ISBN) | BookNew.dc.html |
| 12 | 커뮤니티 피드 | Community.dc.html |
| 13 | 게시글 상세 · 익명 댓글 | PostDetail.dc.html |
| 14 | 글쓰기 | PostNew.dc.html |
| 15 | 모임 목록 | Groups.dc.html |
| 16 | 모임 상세 · 가입 신청 | GroupDetail.dc.html |
| 17 | 시간표 | Timetable.dc.html |
| 18 | 마이 · 매너온도 | MyPage.dc.html |
