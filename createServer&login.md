# Tangle 신고함 · 관리자 페이지 설정 정리

## 목적
- 앱 내 신고(버그 · 의견)를 서버에 저장
- 저장된 신고를 관리자 페이지에서 조회 · 상태 관리
- 별도 관리자 서버 없이 앱과 같은 Cloudflare Pages 프로젝트에서 운영 (주소만 분리)

## 구성
| 구분 | 주소 | 역할 |
|---|---|---|
| 앱 | `https://tangles.pages.dev` | 사용자 화면, 신고 보내기 |
| 신고 받기 | `https://tangles.pages.dev/api/report` | 신고 접수 · 저장 (Pages Functions) |
| 관리자 페이지 | `https://tangles.pages.dev/admin` | 신고 목록 조회 · 상태 변경 (관리자 키 필요) |
| 신고함 | D1 데이터베이스 `tangle-reports` | 신고 데이터 보관 |

## 코드 변경 (GitHub push → 자동 배포)
- `functions/api/report.js`: 신고 받기 입구
- `functions/admin/[[path]].js`: 관리자 페이지 입구
- `server/report/`: 신고 검사 · 저장 · 관리자 화면 · 표 정의(`migrations/0001_reports.sql`)
- 앱 신고 주소 변경: 기존 Worker(`tangle-collab…workers.dev`) → `tangles.pages.dev/api/report`

## Cloudflare 대시보드 설정
| 순서 | 위치 | 설정 | 목적 |
|---|---|---|---|
| 1 | Storage & databases → D1 SQLite Database → Create | 이름 `tangle-reports` | 신고 저장소 생성 (타 프로젝트 DB와 분리) |
| 2 | `tangle-reports` → Console | 표 생성 SQL 실행 (`reports`, `report_history`) | 신고 · 상태 이력 저장 구조 생성 |
| 3 | Workers & Pages → `tangles` → Settings → Bindings → Add → D1 database | `REPORTS` = `tangle-reports` | 코드가 신고함에 접근하도록 연결 |
| 4 | Settings → Variables and secrets → Add | Secret `ADMIN_KEY` | 관리자 페이지 접근 키 (값은 git · 문서에 기록하지 않음) |
| 5 | Deployments → All deployments → `⋯` → Retry deployment | 재배포 | 3 · 4번 설정 적용 |

## 적용 확인
- `POST /api/report` (빈 값): 503 → 400 ✅ 신고함 연결 확인
- `GET /admin/api/reports` (키 없음): 401 ✅ 관리자 키 적용 확인
- `/admin` 로그인: 500 → 표 미생성, Console에서 표 생성 SQL 실행 필요
- 남은 확인: 실제 신고 1건 전송 → `/admin` 목록 표시 확인

## 용어
- API: 앱 ↔ 서버 요청 주소 (`/api/report`, `/admin/api/*`)
- SQL: 서버 ↔ DB 명령 언어 (표 생성 · 저장 · 조회)
- 표(table): DB 안의 데이터 보관 단위 (칸 = 항목, 줄 = 데이터 1건)
- 바인딩: 코드 ↔ D1 연결 이름 (`REPORTS`)
- Secret: 암호화 저장 값 (`ADMIN_KEY`, git 미포함)
- Wrangler: Cloudflare 명령어 도구, 신고함 운영에는 불필요

## 표 생성 SQL 예시
- 용도: D1 Console에서 1회 실행 → 표 생성 (재실행해도 안전)
- `CREATE TABLE`: 표 생성 · 칸 정의 / `CREATE INDEX`: 조회 속도용 색인

```sql
CREATE TABLE IF NOT EXISTS inquiries (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  created_at INTEGER NOT NULL,
  name       TEXT    NOT NULL,
  message    TEXT    NOT NULL,
  status     TEXT    NOT NULL DEFAULT 'new'
);
CREATE INDEX IF NOT EXISTS inquiries_created ON inquiries (created_at DESC);
```
- 바인딩: 코드 ↔ D1 연결 이름 (`REPORTS`)
- Secret: 암호화 저장 값 (`ADMIN_KEY`, git 미포함)
- Wrangler: Cloudflare 명령어 도구, 신고함 운영에는 불필요

## API 사용 이유
- 기기 밖으로 데이터 전달 시 필요 (신고 수집 · 함께 편집)
- 앱 → DB 직접 접근 금지 (보안), 서버가 검증 후 저장

## 오류 코드
- 401: 관리자 키 불일치
- 500: 표 미생성 → Console에서 SQL 실행
- 503: 바인딩 · 키 미설정 → 설정 후 재배포

## 다른 앱 적용 순서
D1 생성 → Console SQL 실행 → Bindings 연결 → Secret 등록 → Retry deployment

---

# 🔐 Tangle 로그인(계정) 설정 정리

## 🎯 목적
- 💳 결제한 사용자 식별 → 계정에 Pro 기록 저장
- 🚪 앱 실행 시 구글 로그인 필수 (DEC_041, 서버 준비 완료 시에만 로그인 창 표시)
- 🔄 계정 기준 기기 간 보드 동기화

## 🧭 구성
| 구분 | 위치 | 역할 |
|---|---|---|
| 🟦 구글 로그인 | Google Cloud 프로젝트 `tangle` → OAuth 클라이언트 | 로그인 창 · 사용자 확인증(ID 토큰) 발급 |
| ☁️ 계정 서버 | `https://tangles.pages.dev/api/account` (Pages Functions) | 확인증 검사 · 계정 생성 · 세션 발급 |
| 🗄️ 계정 데이터 | D1 `tangle-reports` (신고함과 같은 DB) | 사용자 · 구독 · 동기화 보드 보관 |

## 🔥 Firebase
- ❌ 미사용: Tangle 로그인은 Firebase 로그인 아님 → `firebaseConfig` 불필요
- ✅ 설정은 Google Cloud 콘솔에서만 진행

## 🟦 Google Cloud 콘솔 설정
| 순서 | 위치 | 설정 | 목적 | 상태 |
|---|---|---|---|---|
| 1 | 프로젝트 선택 | `tangle` | 로그인 설정 대상 프로젝트 | ✅ |
| 2 | Google 인증 플랫폼 → 브랜딩(동의 화면) | 앱 이름 `Tangle` · 지원 이메일 · 대상 외부 | 로그인 창에 표시되는 앱 정보 | ✅ |
| 3 | Google 인증 플랫폼 → 클라이언트 → 만들기 | 유형 웹 애플리케이션 · 이름 `tangle` | 앱 이름표(클라이언트 ID) 발급 | ✅ |
| 4 | 클라이언트 → 승인된 JavaScript 원본 | `https://tangles.pages.dev` | 이 주소에서만 로그인 버튼 허용 | ✅ |
| 5 | 클라이언트 → 승인된 리디렉션 URI | 비움 | 미사용 | ✅ |
| 6 | Google 인증 플랫폼 → 대상 → 앱 게시 | 테스트 중 → 프로덕션 | 등록한 테스트 사용자 외 누구나 로그인 | ⏳ |

- 🔑 클라이언트 ID: `숫자-영문.apps.googleusercontent.com` → Cloudflare `GOOGLE_CLIENT_IDS`에 입력
- 🚫 클라이언트 보안 비밀번호(Client secret): 미사용 · 어디에도 입력 금지

## ☁️ Cloudflare 대시보드 설정
| 순서 | 위치 | 설정 | 목적 | 상태 |
|---|---|---|---|---|
| 1 | D1 `tangle-reports` → Console | 계정 표 생성 SQL 실행 (`users`, `subscriptions`, `boards`, `guest_links`) | 계정 · 구독 · 동기화 저장 구조 | ⏳ |
| 2 | `tangles` → Settings → Bindings → Add → D1 database | 🔗 `ACCOUNTS` = `tangle-reports` | 로그인 코드가 DB에 접근 (같은 DB에 이름표 추가) | ⏳ |
| 3 | Settings → Variables and secrets → Add | 📝 Text `GOOGLE_CLIENT_IDS` = 클라이언트 ID | 구글 확인증 검사 기준 | ⏳ |
| 4 | Settings → Variables and secrets → Add | 🔒 Secret `SESSION_SECRET` = 40자 이상 무작위 값 | 로그인 유지 표(세션) 서명 (값은 git · 문서에 기록하지 않음) | ⏳ |
| 5 | Settings → Variables and secrets → Add | 🔛 Text `PLANS_ON` = `true` | 계정 서버 켜기 | ⏳ |
| 6 | Deployments → All deployments → `⋯` → Retry deployment | 🚀 재배포 | 1~5번 설정 적용 | ⏳ |

### 🗄️ 계정 표 생성 SQL
```sql
CREATE TABLE IF NOT EXISTS users (id TEXT PRIMARY KEY, email TEXT NOT NULL, name TEXT NOT NULL DEFAULT '', created_at INTEGER NOT NULL, age_ok_at INTEGER NOT NULL, trial_used INTEGER NOT NULL DEFAULT 0);
CREATE TABLE IF NOT EXISTS subscriptions (user_id TEXT PRIMARY KEY, source TEXT NOT NULL, ref TEXT NOT NULL, status TEXT NOT NULL, until INTEGER NOT NULL, updated_at INTEGER NOT NULL);
CREATE TABLE IF NOT EXISTS boards (user_id TEXT NOT NULL, board_id TEXT NOT NULL, data TEXT NOT NULL, updated_at INTEGER NOT NULL, PRIMARY KEY (user_id, board_id));
CREATE TABLE IF NOT EXISTS guest_links (guest_id TEXT PRIMARY KEY, user_id TEXT NOT NULL, linked_at INTEGER NOT NULL);
CREATE INDEX IF NOT EXISTS guest_links_user ON guest_links (user_id);
```

## 🔗 바인딩 현황 (DB 1개 · 이름표 2개)
| 이름 | DB | 쓰는 기능 |
|---|---|---|
| `REPORTS` | `tangle-reports` | 🚩 신고 · 관리자 페이지 |
| `ACCOUNTS` | `tangle-reports` | 🔐 로그인 · 구독 · 동기화 |

## ✅ 적용 확인
- 🩺 `GET /api/account/config`: 현재 `off` (계정 서버 꺼짐) → 설정 후 정상 응답 확인
- 🚪 `https://tangles.pages.dev` 접속 → 구글 로그인 창 표시 확인
- ⚠️ 재배포 즉시 운영판 전체 사용자 로그인 필수

## 🚨 상태별 원인
- `off`: `PLANS_ON` 미설정
- `not-ready`: `ACCOUNTS` · `SESSION_SECRET` · `GOOGLE_CLIENT_IDS` 중 누락
- 로그인 창 오류(origin 불허): 승인된 JavaScript 원본에 `https://tangles.pages.dev` 누락
- 일부 사용자만 로그인 가능: 앱 게시 미완료 (테스트 중 상태)
