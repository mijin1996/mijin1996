# 프로젝트 작업 규칙

> 게임(Unity, C#)과 웹앱(Next.js, TypeScript) 공통 규칙.
> 프로젝트 루트에 `CLAUDE.md`로 두면 Claude Code가 매 작업마다 읽는다.

---

## 0. 작업 순서 (매번 지킬 것)

1. 작업 전 `docs/file-tree.md`, `docs/architecture.md`, `docs/implemented.md`를 읽고 현재 구조를 파악한다.
2. 새 파일을 만들기 전에 비슷한 역할의 파일이 이미 있는지 확인한다. 있으면 확장한다.
3. 구현 → 테스트 작성 → 테스트·린트·빌드 통과 확인 → 문서 갱신 → 커밋 순서로 진행한다.
4. 테스트가 실패하면 다음 단계로 넘어가지 않는다.

---

## 1. 코드 스타일: K&R

- 여는 중괄호 `{`는 선언/조건문과 **같은 줄**에 둔다.
- `else`, `catch`, `finally`는 닫는 중괄호 `}`와 **같은 줄**에 둔다.
- 한 줄짜리 `if`/`for`도 중괄호를 생략하지 않는다.
- 들여쓰기는 스페이스 4칸.

```csharp
public void Jump() {
    if (isGrounded) {
        velocity.y = jumpForce;
    } else {
        bufferedJump = true;
    }
}
```

C#은 기본이 Allman 스타일이므로 `.editorconfig`에 아래를 넣어 강제한다.

```ini
[*.cs]
indent_style = space
indent_size = 4
csharp_new_line_before_open_brace = none
csharp_new_line_before_else = false
csharp_new_line_before_catch = false
csharp_new_line_before_finally = false
```

TypeScript는 Prettier 기본 설정이 K&R과 같으므로 Prettier를 따른다.

---

## 2. 설계 원칙

### SOLID
| 원칙 | 의미 | 적용 방법 |
|---|---|---|
| SRP | 클래스는 변경 이유가 하나뿐이어야 한다 | 이동·입력·애니메이션·점수를 한 클래스에 넣지 않는다 |
| OCP | 확장에는 열려 있고 수정에는 닫혀 있다 | 새 장애물·아이템은 기존 코드 수정 없이 새 클래스/데이터 추가로 만든다 |
| LSP | 하위 타입은 상위 타입을 대체할 수 있어야 한다 | 인터페이스 구현체가 예외를 던지거나 기능을 빼먹지 않는다 |
| ISP | 쓰지 않는 메서드에 의존하지 않는다 | 큰 인터페이스 하나 대신 작은 인터페이스 여러 개 |
| DIP | 구체가 아니라 추상에 의존한다 | 저장소·광고·네트워크는 인터페이스로 받고 구현은 주입한다 |

### 기타 원칙
| 원칙 | 의미 | 적용 방법 |
|---|---|---|
| DRY | 같은 지식을 두 번 쓰지 않는다 | 같은 로직이 두 번째 나오면 공통 함수/클래스로 뺀다 |
| PIE | 의도를 드러내게 작성한다 (Program Intently and Expressively) | 이름만 보고 역할을 알 수 있게 짓고, 영리한 한 줄보다 읽히는 세 줄을 쓴다 |
| SDP | 안정적인 쪽으로 의존한다 | 자주 바뀌는 UI·연출이 잘 안 바뀌는 Core에 의존한다. 반대는 금지 |
| ADP | 모듈 의존에 순환이 없어야 한다 | A→B→A 같은 순환 참조가 생기면 인터페이스를 Core로 올려서 끊는다 |

### 계층과 의존 방향

```
UI / Presentation  ─┐
                    ├──▶  Application(게임 흐름, 유스케이스)  ──▶  Core(도메인 규칙, 순수 C#/TS)
Infrastructure  ────┘       (저장, 광고, 플랫폼 SDK, API — Core의 인터페이스를 구현)
```

- **Core**는 Unity API·React·네트워크를 모른다. 그래서 테스트가 쉽다.
- Unity는 **Assembly Definition(.asmdef)**으로 계층을 나눠서 잘못된 방향의 참조를 컴파일 에러로 막는다 (SDP·ADP 강제).
- 웹앱은 `src/core`, `src/features`, `src/infra`, `src/app`(라우트)로 나누고, `core`는 다른 폴더를 import하지 않는다.

---

## 3. 패턴 사용 규칙

- 패턴은 **문제가 있을 때만** 쓴다. 이름만 멋있는 패턴 추가 금지.
- 패턴을 쓴 클래스/파일 맨 위에 **패턴 이름과 이유를 1줄로** 적는다.

```csharp
// 패턴: Object Pool — 장애물을 매번 생성/파괴하면 GC 스파이크로 프레임이 끊겨서
public class ObstaclePool { ... }
```

```ts
// 패턴: Repository — 리뷰 데이터 출처(CSV/API)가 바뀌어도 화면 코드를 고치지 않으려고
export interface ReviewRepository { ... }
```

### 자주 쓰는 패턴 (게임)
| 패턴 | 쓰는 곳 |
|---|---|
| State | 플레이어 상태(달리기/점프/슬라이드/사망), 게임 상태(대기/플레이/일시정지/게임오버) |
| Object Pool | 장애물, 코인, 이펙트처럼 자주 생기고 사라지는 오브젝트 |
| Observer / 이벤트 | 점수 변화 → UI 갱신, 사망 → 게임오버 화면 (직접 참조 대신) |
| ScriptableObject 데이터 | 속도, 난이도 곡선, 아이템 수치 같은 밸런스 값 |
| Strategy | 장애물 패턴, 난이도 증가 방식처럼 교체 가능한 규칙 |
| Factory | 종류별 장애물/아이템 생성 |

### 자주 쓰는 패턴 (웹앱)
| 패턴 | 쓰는 곳 |
|---|---|
| Repository | 데이터 읽기/쓰기 (CSV, DB, 외부 API) |
| Custom Hook | 컴포넌트에서 상태·데이터 로직 분리 |
| Adapter | 외부 API(Claude API 등) 응답을 내부 타입으로 변환 |

### 금지
- `class` 하나에 모든 기능 몰아넣기 (God Object)
- 싱글톤 남발 (정말 전역이어야 하는 것만, 이유를 주석으로)
- `Update()` 안에서 `GetComponent`, `Find`, `new` 호출

---

## 4. Result 패턴

실패할 수 있는 작업은 예외를 던지지 않고 **Result**를 반환한다.
- 사용처: 저장/불러오기, 파싱, 네트워크, 광고·결제, 플랫폼 SDK 호출
- 예외는 프로그래머 실수(null 인자, 잘못된 상태)에만 쓴다.
- 호출한 쪽은 실패를 반드시 처리한다. Result를 무시하지 않는다.

```csharp
public readonly struct Result<T> {
    public bool IsSuccess { get; }
    public T Value { get; }
    public string Error { get; }

    private Result(bool ok, T value, string error) {
        IsSuccess = ok;
        Value = value;
        Error = error;
    }

    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(string error) => new(false, default, error);
}
```

```ts
export type Result<T, E = string> =
    | { ok: true; value: T }
    | { ok: false; error: E };
```

Result 타입은 프로젝트에 **한 곳에만** 정의한다 (Core 계층).

---

## 5. 테스트와 Mock 데이터

### 테스트
- 기능을 구현하면 테스트를 함께 작성한다.
- 게임: Core 로직은 **EditMode** 테스트, 씬·물리·입력이 필요한 것은 **PlayMode** 테스트 (Unity Test Framework).
- 웹앱: **Vitest + React Testing Library**. `npm test`로 실행.
- 테스트 이름: `메서드_조건_기대결과` (예: `AddScore_CoinCollected_IncreasesBy10`)
- 정상 경로뿐 아니라 실패 경로(Result.Fail)도 테스트한다.

### Mock 데이터
- 테스트는 실제 네트워크·실제 저장소·실제 광고 SDK를 쓰지 않는다. 인터페이스의 Mock 구현을 주입한다.
- Mock은 한 폴더에 모은다.
  - 게임: `Assets/Tests/Mocks/`
  - 웹앱: `src/__mocks__/` 또는 `tests/mocks/`
- Mock 데이터는 팩토리 함수로 만든다 (예: `CreateMockPlayerData(score: 100)`). 테스트마다 같은 데이터를 복붙하지 않는다.

---

## 6. 중복 금지

- 같은 역할의 파일을 두 개 만들지 않는다. 새 파일 전에 `docs/file-tree.md`를 확인한다.
- 같은 로직이 두 번째로 나오면 공통화한다.
- 숫자·문자열 상수는 한 곳에 정의한다 (매직 넘버 금지). 게임 밸런스 값은 ScriptableObject로.
- 안 쓰는 파일·코드·주석 처리된 옛 코드는 지운다. 기록은 git이 한다.

---

## 7. 문서 구조 (`docs/`)

```
docs/
├── file-tree.md          # 전체 파일 구조 + 각 파일 역할 1줄
├── architecture.md       # 계층, 의존 방향, 사용 중인 패턴과 이유
├── prototype.md          # 핵심 플레이 루프/핵심 기능, 프로토타입 범위, 검증할 것
├── implemented.md        # 구현 완료된 기능 목록 (완료일 포함)
├── planned.md            # 구현 예정 기능 목록 (우선순위 순)
└── upload/
    ├── apps-in-toss.md         # 앱인토스
    ├── itch-io.md              # itch.io
    ├── game-jolt.md            # Game Jolt
    ├── google-play-console.md  # Google Play Console
    └── macos.md                # Mac에서 빌드·업로드하는 방법 및 macOS 빌드 배포
```

### 문서 갱신 규칙
- 파일을 추가·삭제·이동하면 `file-tree.md`를 바로 갱신한다.
- 기능이 완료되면 `planned.md`에서 지우고 `implemented.md`로 옮긴다.
- 구조나 패턴이 바뀌면 `architecture.md`를 갱신한다.
- 문서는 위 목록에 있는 파일에만 쓴다. 비슷한 문서를 새로 만들지 않는다.

### 애매한 내용
나에게 다시 물어봐서 확실히 방향을 정하도록 해(임의로  방향을 정하지말기!)

### 메모리 할당을 최대한 최소화하도록 
예를들어, 코루틴 보다 awaitable || Unitask를 사용하여 코드가 가볍게 작성되도록



### `file-tree.md` 형식
```
Assets/Scripts/Core/Score/ScoreCalculator.cs   — 점수 계산 규칙 (순수 C#)
Assets/Scripts/Gameplay/Player/PlayerJump.cs   — 점프 입력 처리와 물리 적용

```

### `upload/*.md` 공통 목차
1. 준비물 (계정, 수수료, 필요한 파일)
2. 빌드 설정 (플랫폼, 해상도, 압축 등)
3. 업로드 절차 (단계별)
4. 심사·공개 설정
5. 체크리스트
6. 자주 나는 오류와 해결법
7. 마지막 확인 날짜와 참고한 공식 문서 링크

업로드 절차는 플랫폼 정책이 자주 바뀌므로, 작성·수정할 때 반드시 **공식 문서를 확인**하고 확인 날짜를 적는다.

---

## 8. 커밋

- 스펙(기능) 하나가 끝나고 테스트·린트·빌드가 모두 통과하면 커밋한다.
- 형식: `feat(범위): 요약`, `fix(범위): 요약`, `refactor(범위): 요약`, `test(범위): 요약`, `docs: 요약`
- 커밋 후 무엇을 했는지 짧게 보고하고 다음 작업으로 넘어간다.

---

## 9. 완료 전 체크리스트

- [ ] K&R 스타일을 지켰다
- [ ] 클래스 하나가 한 가지 일만 한다 (SRP)
- [ ] 의존 방향이 규칙대로이고 순환이 없다 (SDP, ADP)
- [ ] 패턴을 쓴 곳에 이유 1줄 주석이 있다
- [ ] 실패 가능한 작업은 Result를 반환하고, 호출부에서 처리한다
- [ ] 테스트를 작성했고 모두 통과한다 (Mock 사용)
- [ ] 중복 파일·중복 코드·매직 넘버가 없다
- [ ] `docs/` 문서를 갱신했다
- [ ] 커밋했다
