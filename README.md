# Harness Framework

문서를 가드레일 삼아 Claude 세션을 여러 step으로 나눠 순차 실행하는 프로젝트 템플릿.

**대상 스택**: React 또는 Next.js + TypeScript. 훅과 검증 게이트가 npm 스크립트(`lint`/`build`/`test`)를 전제한다.

## 전제조건

| 도구 | 용도 |
|---|---|
| `jq` | 훅 4종이 전부 의존한다 (없으면 매 훅 호출마다 에러) |
| Git Bash (Windows) | 훅이 bash 스크립트다 |
| `python3` | `scripts/execute.py` |
| Claude Code CLI | `execute.py`가 `claude -p`로 호출한다 |

## 시작하기

이 저장소는 GitHub **Template repository**다. `clone`하지 말고 **Use this template**으로 새 저장소를 만든다 — `clone`하면 템플릿 히스토리와 `origin` 리모트가 따라와서, 새 프로젝트 코드를 실수로 템플릿 저장소에 push할 수 있다.

```bash
# 1. 환경변수
cp .env.example .env.local   # 값을 채운다. .env.local 은 커밋되지 않는다.

# 2. 문서 채우기 — 전부 {플레이스홀더} 상태다
#    CLAUDE.md, docs/PRD.md, docs/ARCHITECTURE.md, docs/ADR.md, docs/UI_GUIDE.md
#    이 문서들이 매 step 프롬프트에 가드레일로 주입되므로 반드시 먼저 채운다.

# 3. Claude 세션에서 step 설계
/harness

# 4. 실행
python3 scripts/execute.py {task-name}          # 순차 실행
python3 scripts/execute.py {task-name} --push   # 실행 후 push
```

## 구성

| 경로 | 역할 |
|---|---|
| `CLAUDE.md` | 프로젝트 규칙. 매 step 프롬프트에 주입된다. |
| `docs/` | PRD·ARCHITECTURE·ADR·UI_GUIDE. 매 step 프롬프트에 주입된다. |
| `.claude/commands/harness.md` | `/harness` — step 설계 워크플로우 |
| `.claude/commands/review.md` | `/review` — 변경사항 리뷰 체크리스트 |
| `scripts/execute.py` | step 순차 실행기. 브랜치 생성·자가교정·2단계 커밋 담당. |
| `phases/{task}/` | step 정의와 실행 상태. `/harness`가 생성한다. |

## 훅 (가드레일)

`execute.py`는 Claude를 `--dangerously-skip-permissions`로 실행한다. 권한 프롬프트가 없는 대신 **훅이 유일한 안전장치**다. 훅은 권한 모드와 무관하게 동작한다.

| 스크립트 | Claude | Codex |
|---|---|---|
| `tdd-guard.sh` | PreToolUse `Edit\|Write` | (`codex-tdd-guard.sh`가 위임) |
| `codex-block-dangerous.sh` | PreToolUse `Bash` | PreToolUse `Bash` |
| `codex-tdd-guard.sh` | — | PreToolUse `Edit\|Write\|apply_patch` |
| `codex-verify-gate.sh` | Stop | Stop |

- **tdd-guard** — 테스트 파일 없는 소스 파일 작성을 차단한다. 설정·타입·선언 파일과 프레임워크 진입점(`layout.tsx`, `page.tsx`, `main.tsx` 등)은 면제.
- **block-dangerous** — `rm -rf` / `git push --force` / `git reset --hard` / `DROP TABLE`을 차단한다. **문자열 포함만으로** 걸리므로, 이 패턴을 다루는 명령은 조각내서 조립해야 한다.
- **verify-gate** — 턴 종료 시 `lint` → `build` → `test`를 돌리고, 실패하면 에러 로그를 붙여 자가교정을 유도한다. 한 턴에 한 번만 개입한다.

세 훅 모두 **`package.json`이 없으면 조용히 통과**한다. 스캐폴딩 전에는 강제할 대상이 없기 때문이다.

Codex를 쓰지 않으면 `.codex/hooks.json`은 그냥 비활성 파일로 남는다.

훅 로직은 `scripts/hooks/`의 스크립트에서 고친다. `.claude/settings.json`과 `.codex/hooks.json`은 경로만 가리킨다.

## 주의

**step 0에서 테스트 러너를 반드시 설치할 것.** `create-next-app`과 `create-vite` 둘 다 `test` 스크립트를 만들어 주지 않는다. 없는 채로 두면 verify-gate가 매 턴 `Missing script: "test"`로 실패한다. `/harness`의 step 설계 원칙 8번에 명시돼 있다.
