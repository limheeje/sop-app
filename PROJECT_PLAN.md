# PROJECT PLAN — sop-app

> 이 문서는 프로젝트의 단일 진실 공급원(source of truth)입니다.
> 새 대화를 시작하든, 몇 주 뒤에 돌아오든 이 파일을 먼저 읽으면 전체 맥락이 복원됩니다.
> 결정 사항이 바뀌면 코드가 아니라 **이 문서를 먼저 수정**하고 코드를 맞춥니다.

- 최초 작성일: 2026-07-28
- 상태: 기획 단계 (개발 착수 전)

---

## 1. 프로젝트 개요

**한 줄 정의**: 틱톡형 숏폼 영상 + 커뮤니티(게시판형) 기능을 함께 가진 하이브리드 앱(iOS/AOS 겸용, 웹도 동시 운영).

**MVP 전략**: 영상 스토리지/스트리밍은 초기 비용·난이도가 크므로, **1단계는 텍스트+이미지 기반 커뮤니티 기능으로 먼저 완성**하고, **2단계에서 숏폼 영상 피드를 추가**한다.

**타겟**: 특정 관심사 기반 커뮤니티 (구체적인 주제/타겟은 추후 확정 — 확정되면 이 섹션 업데이트)

---

## 2. 역할 분담

| 영역 | 담당 | 비고 |
|---|---|---|
| 프론트엔드 구현 (HTML/CSS/JS, Next.js) | **사용자** | Next.js 설치/환경설정도 사용자가 직접 진행 |
| 백엔드 설계 및 구현 | **Claude** | API 서버, DB 설계, 인증, 인프라 가이드 |
| 기획 (기능 정의, 우선순위, 로드맵) | **Claude** | 이 문서로 관리 |
| API 설계 | **Claude** | 아래 6번 섹션, 프론트는 이 스펙에 맞춰 연동 |
| 디자인 (톤앤매너, 컴포넌트 가이드) | **Claude** (선택/필수 아님) | 8번 섹션 참고, 실제 CSS 구현은 사용자 |
| 하이브리드 앱 배포 (Capacitor, 스토어 등록) | **가이드는 Claude, 실행은 사용자** | 사용자가 처음이므로 단계별로 안내 |

---

## 3. 기술 스택 (확정)

### Frontend (사용자 담당)
- Next.js (stable, App Router), React 18
- TypeScript 권장 (백엔드와 타입 공유 및 협업 용이)
- 스타일링: 사용자가 직접 HTML/CSS/JS로 작성 (프레임워크 강제 안 함)

### Backend (Claude 담당)
- **Node.js + NestJS + TypeScript**
- **PostgreSQL** + **Prisma ORM**
- 인증: JWT (Access + Refresh Token), 추후 소셜 로그인(카카오/구글) 확장
- 이미지 업로드: S3 호환 스토리지 (S3 또는 Cloudflare R2) + Presigned URL 방식
- 영상(2단계): Cloudflare Stream 또는 Mux 후보 (스토리지+트랜스코딩+CDN 자동 처리, 직접 구축보다 초기 비용/난이도 낮음)

### 하이브리드 앱
- **Capacitor** (Ionic) — 기존 웹앱(Next.js)을 iOS/AOS 앱으로 감싸는 방식

### 인프라 (초기 추천, 무료/저가 티어로 시작)
- 프론트 배포: Vercel
- 백엔드 배포: Railway 또는 Render
- DB: Supabase 또는 Neon (Postgres 무료 티어)
- 추후 트래픽 증가 시 AWS/GCP로 이전 검토

### 저장소 구조 (확정)
```
sop-app/               ← 루트 = Next.js 프로젝트 (사용자 작업 영역)
├── app/ (or pages/)
├── public/
├── server/            ← NestJS (Claude 작업 영역, 루트 안에 위치)
├── PROJECT_PLAN.md
└── README.md
```
`create-next-app`을 루트에서 바로 실행해 프론트를 루트에 두고, 그 안에 `server/` 폴더로 백엔드를 분리한다. 모노레포 툴(Turborepo 등)은 지금 단계에서는 불필요한 복잡도라 판단 — 폴더만 분리.

> 주의: `server/`는 별도의 Node 프로젝트(자체 `package.json`, `node_modules`)이므로 루트의 `.gitignore`에 `server/node_modules`가 포함되는지 확인. Next.js의 `next.config`나 빌드 과정이 `server/`를 프론트 소스로 오인해 스캔하지 않도록 주의(기본적으로는 문제 없음, `app/`이나 `pages/` 밖은 Next가 건드리지 않음).

---

## 4. 하이브리드 앱 전략 (Capacitor) — 초보자용 단계 가이드

### 핵심 개념
Capacitor는 이미 만든 웹사이트(Next.js)를 **그대로 앱 안의 웹뷰에서 실행**시켜주는 도구다. 코드베이스 하나로 웹 + iOS + AOS를 동시에 대응할 수 있다. 단, **웹뷰만 있는 앱은 스토어 심사에서 리젝될 위험**이 있으므로, 최소한의 네이티브 기능(푸시 알림, 공유하기, 스플래시 화면 등)을 반드시 추가한다.

### 단계별 순서 (실행은 사용자, 막히면 Claude에게 요청)

1. **웹사이트 먼저 완성 & 배포** — Next.js 앱을 Vercel에 배포해서 실제 URL로 정상 작동 확인. (Capacitor 세팅은 이게 끝난 뒤 시작)
2. **Capacitor 프로젝트 초기화** — `npm install @capacitor/core @capacitor/cli` → `npx cap init`
3. **iOS/Android 플랫폼 추가** — `npx cap add ios`, `npx cap add android`
4. **`capacitor.config.ts` 설정** — `server.url`을 배포된 Vercel 주소로 지정 (Next.js를 정적 export하지 않고, 라이브 웹사이트를 그대로 로드하는 방식 — SSR/API Route를 그대로 쓸 수 있어 초보자에게 가장 단순함). 이후 웹사이트를 업데이트하면 앱 재배포 없이도 앱 내용이 즉시 갱신됨 (스토어 재심사 불필요, 단 네이티브 코드 변경 시엔 재심사 필요).
5. **네이티브 기능 최소 1~2개 추가** — 예: `@capacitor/push-notifications`, `@capacitor/share`, `@capacitor/splash-screen`. 심사 통과율을 위해 필수.
6. **아이콘/스플래시 생성** — `@capacitor/assets` 패키지로 자동 생성.
7. **개발 환경 준비**
   - iOS: macOS + Xcode 필요, Apple Developer Program 가입 (연 $99)
   - Android: Android Studio 필요, Google Play Developer 계정 (1회 $25)
   - **macOS가 없다면 iOS 빌드가 불가능** — 이 경우 Mac 대여 서비스(MacStadium, Codemagic 등) 또는 클라우드 CI(예: Codemagic, EAS-like 서비스) 검토 필요. 사용자 환경 확인되면 이 섹션 구체화.
8. **내부 테스트** — iOS는 TestFlight, Android는 Play Console 내부 테스트 트랙
9. **스토어 제출 및 심사**

> Phase 3 로드맵(7번 섹션)에서 이 단계들을 다시 체크리스트로 정리함.

---

## 5. 백엔드 아키텍처

### NestJS 모듈 구성 (1단계 커뮤니티 MVP 기준)
- `AuthModule` — 회원가입/로그인/토큰 재발급
- `UserModule` — 프로필, 팔로우/팔로워
- `PostModule` — 게시글 CRUD, 피드
- `CommentModule` — 댓글/대댓글
- `LikeModule` — 좋아요
- `MediaModule` — 이미지 업로드 (presigned URL 발급)
- `NotificationModule` — 알림 (팔로우/좋아요/댓글 발생 시)

2단계 추가: `VideoModule` (숏폼 영상 업로드/피드/조회수)

### 데이터 모델 개요 (Prisma 스키마 초안)

```
User
 - id, email, passwordHash, nickname, profileImageUrl, bio, createdAt

Post
 - id, authorId(→User), content, imageUrls[], createdAt, updatedAt

Comment
 - id, postId(→Post), authorId(→User), parentId(→Comment, nullable), content, createdAt

Like
 - id, userId(→User), postId(→Post), createdAt  (unique: userId+postId)

Follow
 - id, followerId(→User), followingId(→User), createdAt (unique: followerId+followingId)

Notification
 - id, recipientId(→User), actorId(→User), type(FOLLOW|LIKE|COMMENT), postId(nullable), isRead, createdAt
```

세부 필드/인덱스는 실제 Prisma `schema.prisma` 작성 시 확정 (개발 착수 시 진행).

---

## 6. API 설계 초안 (REST, `/api/v1` prefix)

### 공통 규칙
- 인증: `Authorization: Bearer <accessToken>`
- 응답 포맷: `{ "data": ..., "meta": ... }` (성공), `{ "error": { "code", "message" } }` (실패)
- 목록 조회: 커서 기반 페이지네이션 (`?cursor=&limit=`) — 무한 스크롤 피드에 적합

### Auth
| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/auth/signup` | 회원가입 |
| POST | `/auth/login` | 로그인 → accessToken + refreshToken |
| POST | `/auth/refresh` | 토큰 재발급 |
| POST | `/auth/logout` | 로그아웃 |

### User
| Method | Endpoint | 설명 |
|---|---|---|
| GET | `/users/me` | 내 프로필 |
| PATCH | `/users/me` | 프로필 수정 |
| GET | `/users/:id` | 특정 유저 프로필 |
| POST | `/users/:id/follow` | 팔로우 |
| DELETE | `/users/:id/follow` | 언팔로우 |
| GET | `/users/:id/followers` | 팔로워 목록 |
| GET | `/users/:id/following` | 팔로잉 목록 |

### Post (피드)
| Method | Endpoint | 설명 |
|---|---|---|
| GET | `/posts` | 피드 조회 (커서 페이지네이션) |
| POST | `/posts` | 게시글 작성 |
| GET | `/posts/:id` | 게시글 상세 |
| DELETE | `/posts/:id` | 게시글 삭제 |
| POST | `/posts/:id/like` | 좋아요 |
| DELETE | `/posts/:id/like` | 좋아요 취소 |

### Comment
| Method | Endpoint | 설명 |
|---|---|---|
| GET | `/posts/:id/comments` | 댓글 목록 |
| POST | `/posts/:id/comments` | 댓글 작성 |
| DELETE | `/comments/:id` | 댓글 삭제 |

### Media
| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/media/presign` | 업로드용 presigned URL 발급 → 프론트가 그 URL로 S3에 직접 업로드 (서버 부하 최소화) |

### Notification
| Method | Endpoint | 설명 |
|---|---|---|
| GET | `/notifications` | 알림 목록 |
| PATCH | `/notifications/:id/read` | 읽음 처리 |

> 실제 구현 시작 시 요청/응답 상세 스펙(Swagger)을 `server/` 안에 코드로 관리하고, 이 문서에는 개요만 유지.

---

## 7. 로드맵

### Phase 1 — 기반 다지기
- [ ] (사용자) 루트에 Next.js 프로젝트 생성 (진행 중)
- [ ] (Claude) 루트 안 `server/` 폴더에 NestJS 프로젝트 스캐폴딩 + Auth 모듈
- [ ] (Claude) Prisma 스키마 확정 및 마이그레이션
- [ ] (공통) 로컬에서 프론트-백엔드 연동 테스트 (로그인 플로우)

### Phase 2 — 커뮤니티 코어 기능
- [ ] 피드 (게시글 목록, 무한 스크롤)
- [ ] 게시글 작성/삭제, 이미지 업로드
- [ ] 댓글/좋아요/팔로우
- [ ] 프로필 페이지
- [ ] 알림

### Phase 3 — 하이브리드 앱 전환
- [ ] 웹사이트 배포 안정화 (Vercel)
- [ ] Capacitor 세팅 (4번 섹션 1~6단계)
- [ ] 개발자 계정 준비 (Apple/Google)
- [ ] 내부 테스트 (TestFlight/Play Console)
- [ ] 스토어 제출

### Phase 4 — 숏폼 영상 기능 (추후)
- [ ] 영상 스토리지/스트리밍 서비스 선정 (Cloudflare Stream vs Mux)
- [ ] 영상 업로드/트랜스코딩 파이프라인
- [ ] 영상 피드 (세로 스와이프 UI는 프론트 담당)
- [ ] 조회수/시청 완료율 등 지표

---

## 8. 디자인 가이드 (선택 — 필요 시 참고)

- 톤앤매너: 다크 모드 기본, 높은 대비, 미디어(이미지/영상) 중심 레이아웃 — 커뮤니티+숏폼 서비스에서 공통적으로 검증된 방향
- 구체적인 컬러 팔레트/컴포넌트 스펙이 필요해지면 요청 시 별도로 작성 (dataviz/artifact 도구로 시안 제작 가능)

---

## 9. 다음 액션 아이템

1. (사용자) 루트에서 `create-next-app`으로 Next.js 프로젝트 생성 (진행 중)
2. (Claude) 루트 안 `server/` 폴더에 NestJS 프로젝트 생성 + Auth 모듈부터 구현 시작
3. (공통) Phase 1 완료 후 Phase 2 착수

> 진행하면서 결정이 바뀌거나 새로운 제약이 생기면 이 문서를 계속 업데이트합니다.
