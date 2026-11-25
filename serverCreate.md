# Tangle 신고함 · 관리자 페이지 설정 정리

## 1. 목적
- 앱 내 신고(버그 · 의견)를 서버에 저장
- 저장된 신고를 관리자 페이지에서 조회 · 상태 관리
- 별도 관리자 서버 없이 앱과 같은 Cloudflare Pages 프로젝트에서 운영 (주소만 분리)

## 2. 구성
| 구분 | 주소 | 역할 |
|---|---|---|
| 앱 | `https://tangles.pages.dev` | 사용자 화면, 신고 보내기 |
| 신고 받기 | `https://tangles.pages.dev/api/report` | 신고 접수 · 저장 (Pages Functions) |
| 관리자 페이지 | `https://tangles.pages.dev/admin` | 신고 목록 조회 · 상태 변경 (관리자 키 필요) |
| 신고함 | D1 데이터베이스 `tangle-reports` | 신고 데이터 보관 |

## 3. 코드 변경 (GitHub push → 자동 배포)
- `functions/api/report.js`: 신고 받기 입구
- `functions/admin/[[path]].js`: 관리자 페이지 입구
- `server/report/`: 신고 검사 · 저장 · 관리자 화면 · 표 정의(`migrations/0001_reports.sql`)
- 앱 신고 주소 변경: 기존 Worker(`tangle-collab…workers.dev`) → `tangles.pages.dev/api/report`

## 4. Cloudflare 대시보드 설정
| 순서 | 위치 | 설정 | 목적 |
|---|---|---|---|
| 1 | Storage & databases → D1 SQLite Database → Create | 이름 `tangle-reports` | 신고 저장소 생성 (타 프로젝트 DB와 분리) |
| 2 | `tangle-reports` → Console | 표 생성 SQL 실행 (`reports`, `report_history`) | 신고 · 상태 이력 저장 구조 생성 |
| 3 | Workers & Pages → `tangles` → Settings → Bindings → Add → D1 database | `REPORTS` = `tangle-reports` | 코드가 신고함에 접근하도록 연결 |
| 4 | Settings → Variables and secrets → Add | Secret `ADMIN_KEY` | 관리자 페이지 접근 키 (값은 git · 문서에 기록하지 않음) |
| 5 | Deployments → All deployments → `⋯` → Retry deployment | 재배포 | 3 · 4번 설정 적용 |

## 5. 적용 확인
- `POST /api/report` (빈 값): 503(준비 안 됨) → 400(입력 오류) ✅ 신고함 연결 확인
- `GET /admin/api/reports` (키 없음): 401(키 필요) ✅ 관리자 키 적용 확인
- 남은 확인: 실제 신고 1건 전송 → `/admin`에서 목록 표시 확인

## 6. 미적용 (needFix)
- Preview(테스트판 `dev.tangles.pages.dev`) 환경: D1 바인딩 · `ADMIN_KEY` 미설정 → 테스트판 신고 불가
- 함께 편집 서버: 기존 Worker 삭제로 연결 불가, 새 Worker 배포 및 주소 변경 필요 (W-13)
