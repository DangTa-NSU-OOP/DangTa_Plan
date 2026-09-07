# MVP 프론트엔드 가이드

> "프론트엔드가 뭘 만들어야 하는지 모르겠다"에 대한 답.
> 이 문서 하나로 프론트 2명이 무엇을·어떤 순서로 만들지 알 수 있게 정리.

---

## 0. 프론트엔드가 하는 일 (개념)

- **백엔드(Spring Boot)** = 데이터베이스에 저장하고, 규칙을 처리하고, `/api/...` 주소로 **JSON**을 주고받는 서버.
- **프론트엔드** = 사용자가 보는 **화면**. 브라우저에서 돌아가는 프로그램.
  - 사용자가 버튼을 누르면 → 프론트가 백엔드 API를 `fetch`로 호출 → 받은 JSON을 화면에 그림.
  - 예: "상품 목록" 화면을 열면 → `GET /api/products` 호출 → 받은 배열을 카드로 렌더링.
- 즉 프론트가 만드는 것 = **화면(페이지) + 재사용 부품(컴포넌트) + API를 부르는 코드 + 로그인 상태 관리**.

```
[사용자] --클릭--> [프론트엔드 React 앱] --HTTP 요청--> [백엔드 Spring API] --> [DB]
                        ^                                    |
                        +--------- JSON 응답 ----------------+
```

---

## 1. 스택 (이걸로 고정)

| 도구 | 용도 | 왜 |
|---|---|---|
| **Vite + React + TypeScript** | 기본 골격 | 설정 거의 없음, 빠름. Next.js의 SSR/서버컴포넌트 개념 학습 비용 제거 |
| **React Router** (`react-router-dom`) | 화면 간 이동(URL) | SPA 표준 |
| **TanStack Query** (`@tanstack/react-query`) | 서버 데이터 불러오기/캐싱/로딩·에러 상태 | 이거 없으면 `useEffect` 지옥. 러닝커브 있지만 이득이 큼 |
| **Tailwind CSS** | 스타일링 | `docs/06-design-system.md`의 색·간격 토큰을 그대로 클래스로 |
| **zustand** (아주 작게) | 로그인 상태(토큰, 내 정보)만 전역 보관 | Redux는 과함 |
| `fetch` (내장) 또는 `axios` | HTTP | 취향. 아래 예시는 fetch |

> TanStack Query가 부담되면 1~2주차는 커스텀 훅(`useEffect` + `useState`)으로 시작하고, 익숙해지면 갈아타도 된다.

---

## 2. 만들 화면 (라우트) 목록

MVP 탭은 **4개**: 홈(중고거래) · 커뮤니티 · 채팅 · 마이

| # | 경로 | 화면 | 로그인 필요 | 주로 쓰는 API | 와이어프레임 |
|---|---|---|---|---|---|
| 1 | `/login` | 로그인 | X | `POST /auth/login` | Login |
| 2 | `/signup` | 회원가입 (이메일→코드→비번·닉네임 3단계 or 1페이지) | X | `POST /auth/email/send-code`, `POST /auth/signup` | EmailVerify, Onboarding |
| 3 | `/` | **홈 = 상품 피드** (검색·카테고리 칩·카드 목록·글쓰기 버튼) | O | `GET /products`, `GET /categories` | Main |
| 4 | `/products/:id` | 상품 상세 (이미지·판매자·찜·채팅하기) | O | `GET /products/:id`, `POST /products/:id/favorite`, `POST /chat/rooms` | ProductDetail |
| 5 | `/products/new` | 상품 등록 | O | `POST /uploads/image`, `POST /products` | ProductNew |
| 6 | `/products/:id/edit` | 상품 수정 (5번 재사용) | O | `PATCH /products/:id` | ProductNew |
| 7 | `/community` | 커뮤니티 (게시판 탭 + 글 목록) | O | `GET /boards`, `GET /boards/:slug/posts` | Community |
| 8 | `/community/posts/:id` | 글 상세 + 댓글 | O | `GET /posts/:id`, `GET/POST /posts/:id/comments`, `POST /posts/:id/like` | PostDetail |
| 9 | `/community/write` | 글쓰기 (게시판 선택·익명 토글) | O | `POST /posts` | PostNew |
| 10 | `/chat` | 채팅방 목록 | O | `GET /chat/rooms` | ChatList |
| 11 | `/chat/:roomId` | 채팅방 (폴링) | O | `GET/POST /chat/rooms/:id/messages`, `POST .../read`, `POST .../complete` | ChatRoom |
| 12 | `/my` | 마이 (프로필·매너온도·내 판매/찜 링크) | O | `GET /users/me` | MyPage |
| 13 | `/my/products`, `/my/favorites` | 내 판매내역 / 찜 목록 | O | `GET /users/me/products`, `GET /users/me/favorites` | (Main 재사용) |
| 14 | `/manner/review?productId=&targetId=` | 매너 평가 (모달 또는 페이지) | O | `GET /manner/pending`, `POST /manner/reviews` | MannerReview |
| 15 | `/users/:id` | 다른 사람 프로필 (선택) | O | `GET /users/:id`, `GET /users/:id/manner-reviews` | (MyPage 축약) |

> 화면 디자인은 이미 만들어둔 하이파이 목업(캔버스 Artifact)과 `docs/06-design-system.md`를 그대로 참고. **새로 디자인할 필요 없음**, 목업을 코드로 옮기면 됨.

---

## 3. 공통 컴포넌트 (부품) 목록

`src/components/` 에 만들어 여러 화면에서 재사용.

### 레이아웃
- `AppLayout` — 상단 앱바 + 하단 탭바를 감싸는 틀 (로그인 후 화면 공통)
- `TabBar` — 하단 4탭 (홈·커뮤니티·채팅·마이), 현재 탭 강조, 채팅 안읽음 뱃지
- `AppBar` — 상단 바 (제목/뒤로가기/우측 아이콘 슬롯)
- `AuthLayout` — 로그인/회원가입용 (탭바 없음)

### 기본 UI
- `Button` (variant: primary / ghost / danger, size)
- `TextField` / `TextArea` (라벨·에러 표시)
- `Chip` (선택형, 카테고리·게시판 필터)
- `Avatar` (이미지 없으면 닉네임 첫 글자 + 색)
- `MannerBadge` (`42.3℃` + 미니 게이지, 온도 구간별 색)
- `Modal` / `BottomSheet` (신고 사유 선택, 매너 평가 등)
- `EmptyState` (목록 비었을 때 안내 + 액션)
- `Spinner` / `SkeletonCard` (로딩 표시)
- `ConfirmDialog` (삭제 확인)

### 도메인 컴포넌트
- `ProductCard` — 피드/찜/내판매 목록의 상품 한 줄 (썸네일·제목·가격·상태뱃지·찜수)
- `PostCard` — 커뮤니티 글 목록의 한 줄
- `CommentItem` — 댓글 하나
- `ChatRoomItem` — 채팅 목록의 방 하나
- `MessageBubble` — 채팅 말풍선 (내 것/상대 것)
- `ImageUploader` — 사진 선택 → `POST /uploads/image` → 미리보기 (상품 등록에서)
- `CategoryChips` / `BoardTabs` — 필터 바
- `StatusSelect` — 거래상태 변경 바텀시트 (판매중/예약중/거래완료)
- `ReportButton` — 신고 버튼 + 사유 시트

### 유틸
- `formatPrice(number)` → `"60,000원"` (0이면 `"나눔"`)
- `timeAgo(date)` → `"3분 전"`
- `mannerColor(temp)` → 온도에 따른 색

---

## 4. 폴더 구조

```
web/
├─ index.html
├─ vite.config.ts
├─ tailwind.config.js
├─ .env                      # VITE_API_BASE=http://localhost:8080
├─ src/
│  ├─ main.tsx               # React 진입점, Router·QueryClient 세팅
│  ├─ App.tsx                # 라우트 정의
│  ├─ lib/
│  │  ├─ api.ts              # fetch 래퍼 (baseURL, 토큰 헤더, 401 처리)
│  │  └─ format.ts           # formatPrice, timeAgo ...
│  ├─ store/
│  │  └─ auth.ts             # zustand: token, user, login(), logout()
│  ├─ components/            # 위 3번의 공통 컴포넌트
│  │  ├─ layout/
│  │  └─ ui/
│  ├─ features/              # 도메인별로 묶기
│  │  ├─ auth/               # api 호출 함수 + 화면
│  │  │  ├─ authApi.ts
│  │  │  ├─ LoginPage.tsx
│  │  │  └─ SignupPage.tsx
│  │  ├─ product/
│  │  │  ├─ productApi.ts
│  │  │  ├─ productQueries.ts   # useProducts(), useProduct(id) ...
│  │  │  ├─ FeedPage.tsx
│  │  │  ├─ ProductDetailPage.tsx
│  │  │  └─ ProductFormPage.tsx
│  │  ├─ community/
│  │  ├─ chat/
│  │  └─ manner/
│  └─ pages/
│     └─ MyPage.tsx
```

- **feature 폴더** 안에 그 도메인의 API 함수·쿼리 훅·페이지를 모아두면 2명이 겹치지 않게 작업 가능.

---

## 5. API 연동 (핵심)

### 5.1 fetch 래퍼 (`src/lib/api.ts`)

```ts
const BASE = import.meta.env.VITE_API_BASE; // .env 에서

export async function api<T>(path: string, options: RequestInit = {}): Promise<T> {
  const token = localStorage.getItem("token");
  const res = await fetch(`${BASE}/api${path}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  });

  if (res.status === 401) {
    localStorage.removeItem("token");
    window.location.href = "/login";
    throw new Error("UNAUTHENTICATED");
  }
  const body = await res.json();
  if (!res.ok) throw new Error(body?.error?.code ?? "ERROR");
  return body.data as T;
}

// 사용 예
export const getProducts = (params: string) =>
  api<ProductListResponse>(`/products?${params}`);
export const createProduct = (dto: CreateProductDto) =>
  api<{ id: number }>(`/products`, { method: "POST", body: JSON.stringify(dto) });
```

### 5.2 이미지 업로드는 별도 (multipart)

```ts
export async function uploadImage(file: File): Promise<string> {
  const token = localStorage.getItem("token");
  const form = new FormData();
  form.append("file", file);
  const res = await fetch(`${BASE}/api/uploads/image`, {
    method: "POST",
    headers: { Authorization: `Bearer ${token}` }, // Content-Type 넣지 말 것 (브라우저가 자동)
    body: form,
  });
  const body = await res.json();
  return body.data.url; // "/uploads/xxx.jpg"
}
```
→ 서버가 준 경로 앞에 `VITE_API_BASE`를 붙여서 `<img src>`에 사용.

### 5.3 TanStack Query로 데이터 불러오기 (`productQueries.ts`)

```ts
import { useQuery } from "@tanstack/react-query";

export function useProducts(filters: ProductFilters) {
  const params = new URLSearchParams(filters as any).toString();
  return useQuery({
    queryKey: ["products", filters],
    queryFn: () => getProducts(params),
  });
}
```

```tsx
// FeedPage.tsx
function FeedPage() {
  const [filters, setFilters] = useState({ page: 0, size: 20 });
  const { data, isLoading, isError } = useProducts(filters);

  if (isLoading) return <SkeletonCard count={5} />;
  if (isError) return <EmptyState text="목록을 불러오지 못했어요" />;

  return (
    <div>
      {data.data.map((p) => <ProductCard key={p.id} product={p} />)}
    </div>
  );
}
```

이 패턴(쿼리 훅 → 화면에서 `isLoading/isError/data` 분기)을 **모든 목록·상세 화면에 반복**하면 됨.

---

## 6. 로그인 상태 관리 (`src/store/auth.ts`)

```ts
import { create } from "zustand";

interface AuthState {
  token: string | null;
  user: Me | null;
  setAuth: (token: string, user: Me) => void;
  logout: () => void;
}

export const useAuth = create<AuthState>((set) => ({
  token: localStorage.getItem("token"),
  user: null,
  setAuth: (token, user) => {
    localStorage.setItem("token", token);
    set({ token, user });
  },
  logout: () => {
    localStorage.removeItem("token");
    set({ token: null, user: null });
  },
}));
```

- 앱이 켜지면 토큰이 있을 때 `GET /users/me`를 한 번 호출해 `user`를 채운다.
- **전역 상태는 이것만.** 나머지 데이터(상품·글·채팅)는 전부 TanStack Query가 관리.

### 보호된 라우트

```tsx
function RequireAuth({ children }: { children: ReactNode }) {
  const token = useAuth((s) => s.token);
  if (!token) return <Navigate to="/login" replace />;
  return <>{children}</>;
}

// App.tsx
<Route element={<RequireAuth><AppLayout /></RequireAuth>}>
  <Route path="/" element={<FeedPage />} />
  <Route path="/products/:id" element={<ProductDetailPage />} />
  {/* ... 나머지 로그인 필요한 화면 */}
</Route>
<Route path="/login" element={<LoginPage />} />
<Route path="/signup" element={<SignupPage />} />
```

---

## 7. 폴링 채팅 구현

```tsx
function ChatRoomPage() {
  const { roomId } = useParams();
  const [messages, setMessages] = useState<Message[]>([]);
  const lastIdRef = useRef(0);

  useEffect(() => {
    let alive = true;
    async function poll() {
      const newMsgs = await api<Message[]>(
        `/chat/rooms/${roomId}/messages?afterId=${lastIdRef.current}`
      );
      if (alive && newMsgs.length) {
        setMessages((prev) => [...prev, ...newMsgs]);
        lastIdRef.current = newMsgs[newMsgs.length - 1].id;
        await api(`/chat/rooms/${roomId}/read`, {
          method: "POST",
          body: JSON.stringify({ lastMessageId: lastIdRef.current }),
        });
      }
    }
    poll();                                   // 첫 로드
    const timer = setInterval(poll, 4000);    // 4초마다
    return () => { alive = false; clearInterval(timer); };
  }, [roomId]);

  // 메시지 전송 후에도 poll() 한 번 즉시 호출하면 딜레이 없이 보임
}
```

- 채팅 목록(`/chat`)도 화면이 열려 있는 동안 10초마다 새로고침하면 충분.
- TanStack Query의 `refetchInterval` 옵션으로도 가능.

---

## 8. FE-1 / FE-2 분담

| | FE-1 | FE-2 |
|---|---|---|
| 담당 화면 | 로그인·회원가입, 홈 피드, 상품 상세, 상품 등록/수정, 마이(프로필) | 커뮤니티(목록·상세·글쓰기), 채팅(목록·방), 매너 평가, 다른 사람 프로필 |
| 공통 작업 | **AppLayout·TabBar·AppBar·Button·TextField·Chip·Avatar·EmptyState·api.ts·auth.ts** 를 먼저 만들어 공유 | ProductCard/PostCard 등 자기 도메인 카드, MessageBubble, MannerBadge |
| 원칙 | 1주차에 FE-1이 공통 뼈대 + 디자인 토큰을 세팅하고 나서 둘이 병렬 | feature 폴더로 분리해 충돌 최소화 |

---

## 9. 셋업 & 배포

```bash
# 생성
npm create vite@latest web -- --template react-ts
cd web
npm i react-router-dom @tanstack/react-query zustand
npm i -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

# .env
echo "VITE_API_BASE=http://localhost:8080" > .env

npm run dev     # http://localhost:5173
npm run build   # dist/ 정적 파일 생성
```

- **CORS**: 백엔드에서 `http://localhost:5173`(개발), 배포 프론트 도메인을 허용해야 함.
- 배포: `dist/`를 Vercel/Netlify에 올리면 끝(정적). 환경변수 `VITE_API_BASE`를 배포된 백엔드 주소로.
- 이미지 경로: 백엔드가 `/uploads/x.jpg`를 주면 프론트에서 `${VITE_API_BASE}/uploads/x.jpg`로 표시.

---

## 10. 주차별 프론트 할 일 (`docs/08` 일정에 맞춤)

| 주차 | FE 작업 |
|---|---|
| 0 | Vite 프로젝트 생성, Tailwind + 디자인 토큰(색·폰트), 라우터·QueryClient 세팅, `api.ts`·`auth.ts`, AppLayout/TabBar 스켈레톤, 백엔드와 "로그인→홈" 연결 확인 |
| 1 | 로그인·회원가입 화면 완성(토큰 저장, RequireAuth), 공통 UI 컴포넌트(Button/TextField/Chip/Avatar/EmptyState) |
| 2 | 홈 피드(목록·검색·카테고리 칩), ProductCard, 상품 상세 |
| 3 | 상품 등록/수정 + ImageUploader, 찜, 상태 변경, 내 판매/찜 목록 |
| 4 | 커뮤니티 목록·상세·댓글·좋아요·글쓰기, PostCard |
| 5 | 채팅 목록·채팅방(폴링), MessageBubble, 안읽음 뱃지 |
| 6 | 매너 평가 화면, 마이페이지(프로필·매너온도·후기), 다른 사람 프로필 |
| 7 | 통합 QA — 로딩·에러·빈 상태 전부 처리, 반응형 점검(모바일 폭), 신고 버튼, 디자인 다듬기 |
| 8 | 배포, 시연 리허설, README |

---

## 11. 안 만들어도 되는 것 (시간 아끼기)

- 무한 스크롤 → **"더 보기" 버튼** 또는 페이지 번호로 충분
- 실시간(WebSocket) → 폴링
- 다크 모드
- PWA / 오프라인
- 이미지 크롭·압축 → 원본 그대로 업로드
- 애니메이션·트랜지션 → 최소한만
- SSR·SEO → CSR이라 신경 안 씀
- 접근성 완벽 대응 → 라벨·버튼 태그 정도만 지키기
- 상태관리 라이브러리 심화 → auth만 zustand, 나머지 Query

---

## 12. 참고 자료

- **MVP 화면 목업**: https://claude.ai/code/artifact/de79815a-90b8-4c4b-b9df-61389f2ed3e3 + `wireframes/mvp/` 폴더 (2번 표의 화면 이름과 파일명이 1:1). 이걸 코드로 옮기면 됨
- (전체 기획 목업 18개는 `wireframes/` 루트 — 참고만)
- 색·폰트·컴포넌트 규칙: `docs/06-design-system.md`
- API 규격: `docs/mvp/api.md`
- 데이터 구조: `docs/mvp/erd.md`
- 전체 범위·일정·팀 분담: `docs/08-mvp-scope.md`
