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
