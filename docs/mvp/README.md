# 당타 MVP 문서

> `docs/08-mvp-scope.md`에서 정한 **학기 프로젝트용 축소 범위**의 실제 설계.
> 전체 기획(`docs/01~07`)이 아니라 **이 폴더만 보고 구현**한다.

## MVP 범위 (한눈에)

| 도메인 | 만드는 것 |
|---|---|
| 인증 | 학교 이메일 + 비밀번호 가입(도메인으로 대학 자동 매칭), 로그인(JWT), 내 프로필 |
| 중고거래 | 상품 등록·목록·상세·수정·삭제, 검색, 카테고리 필터, 거래상태, 찜 |
| 커뮤니티 | 게시판, 글 CRUD, 댓글(1단계), 좋아요, 익명 |
| 채팅 | 상품 문의 1:1 채팅 (실시간 아님 — 3~5초 폴링) |
| 매너온도 | 거래완료 후 태그 평가 → 온도 가감, 프로필 표시 |
| 공통 | 모바일 레이아웃(탭 4개), 신고 버튼, 시드 데이터 |

**제외**: 중고책 별도 탭, 소모임, 시간표, WebSocket, 푸시, S3, Redis, 관리자 화면, 대댓글, 약속잡기, 스크랩, 투표

## 문서

| 파일 | 내용 |
|---|---|
| [erd.md](./erd.md) | 15개 테이블. mermaid ERD + 컬럼 정의 |
| [api.md](./api.md) | REST API 40여 개. 요청/응답 예시 |
| [frontend.md](./frontend.md) | **프론트엔드가 뭘 만드는지** — 화면·컴포넌트·폴더구조·API 연동·상태관리·주차별 할 일 |

**화면 목업**: `wireframes/mvp/` (13개 화면) · 캔버스 https://claude.ai/code/artifact/de79815a-90b8-4c4b-b9df-61389f2ed3e3

## 스택 (축소판)

- **백엔드**: Spring Boot 3 + Spring Web + Spring Data JPA + Spring Security(JWT) + DB(H2로 시작 → MySQL/PostgreSQL)
- **프론트엔드**: Vite + React + TypeScript + React Router + TanStack Query + Tailwind CSS
  - (전체 기획은 Next.js였지만, 학기 프로젝트는 SSR 개념까지 갈 여유가 없어 CSR SPA로 단순화)
- **이미지**: 서버 로컬 폴더 저장 + 정적 서빙 (S3 안 씀)
- **배포**: 백엔드 1대(Render/Railway/학교서버) + 프론트 정적 호스팅(Vercel/Netlify), 또는 로컬 시연
