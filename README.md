# commit-plan-only

Cursor Agent Skill: Changes를 **리뷰하기 좋은 커밋 단위로 쪼개는 계획**만 추천합니다.  
에이전트가 `git add` / `git commit`을 실행하지는 **않습니다** (plan-only).

| | |
|---|---|
| 스킬 이름 | `commit-plan-only` |
| 이 레포 | 단일 스킬 전용 (`SKILL.md`가 루트) |
| 로컬 원본 (개인용) | `~/.cursor/skills/split-commits/SKILL.md` |

이름이 `split-commits`가 아닌 이유: 동명 스킬 중에는 **직접 스테이징·커밋**하는 것도 있어서, **plan-only**임을 구분합니다.

---

## 목차

1. [이 스킬은 언제 쓰나](#1-이-스킬은-언제-쓰나)
2. [어떻게 구성되어 있나](#2-어떻게-구성되어-있나)
3. [어떻게 쓰나](#3-어떻게-쓰나)
4. [에이전트가 내는 결과물](#4-에이전트가-내는-결과물)
5. [설치 방법](#5-설치-방법)
6. [배포·유지보수](#6-배포유지보수)

---

## 1. 이 스킬은 언제 쓰나

### 맞는 경우

작업 트리(Changes)가 커져서 **한 번에 커밋하기엔 부담**인데, 에이전트에게 커밋까지 맡기기는 싫을 때 씁니다.

예를 들어:

- “커밋 어떻게 쪼개면 돼?”
- “commit plan / commit 추천 / 커밋 쪼개기”
- “Changes를 리뷰하기 좋게 나눠줘”
- “순서랑 `git add` 경로만 알려줘, 커밋은 내가 할게”

에이전트는 **읽기 전용**으로 `git status` / `diff` / `log`만 보고, 복사해서 쓸 수 있는 스테이징 명령과 메시지·순서 계획을 줍니다. **스테이징·커밋·푸시는 사용자가 직접** 합니다.

### 맞지 않는 경우

| 원하는 것 | 이 스킬 대신 |
|-----------|----------------|
| “이대로 커밋해줘” / 실제로 `git commit` 실행 | 일반 커밋 요청 (이 스킬 밖). 사용자 규칙의 committing 절차 따름 |
| PR 생성, push, amend | 해당 작업용 일반 지시 |
| 코드 리뷰·버그 수정 | 리뷰/수정용 스킬 또는 일반 채팅 |

**핵심:** 기본 동작은 **계획만 내고 끝**입니다. “이대로 커밋해 줄까요?”를 CTA로 붙이지 않습니다. 나중에 별도로 “커밋해줘”라고 하면, 그때는 이 스킬 밖 일반 커밋 규칙을 따릅니다.

---

## 2. 어떻게 구성되어 있나

### 2.1 이 레포 (단일 스킬)

```text
commit-plan-only/              ← Git 저장소 루트 (= 이 폴더)
├── README.md                  ← 이 파일
├── SKILL.md                   ← 스킬 본체 (frontmatter name = commit-plan-only)
└── .gitignore
```

여러 스킬을 담는 `skills/<name>/` 카탈로그 레포가 **아닙니다**.  
이 저장소 전체가 `commit-plan-only` 하나입니다. `SKILL.md`는 루트에 둡니다.

### 2.2 `SKILL.md` 내부 구조

에이전트가 따르는 지시서는 YAML frontmatter + 마크다운 본문입니다.

```markdown
---
name: commit-plan-only          # 설치·식별자
description: >-                 # Cursor가 “이 스킬을 쓸지” 고를 때 쓰는 트리거 문장
  … commit 쪼개기, commit plan, …
---

# Commit plan only (recommend only)
## When to use / Workflow / Split rules / …
## Shell-safe git add (required)
## Output template
## Hard rules
```

| 구역 | 역할 |
|------|------|
| `name` | `npx skills add …` / Cursor가 쓰는 식별자 |
| `description` | “커밋 쪼개기”, “commit plan” 등 사용자 말에 반응하는 트리거 |
| Workflow | `status` / `diff` / `log`를 **병렬·읽기 전용**으로 모은 뒤 클러스터링 |
| Split rules | 관심사 1커밋, 수직 슬라이스, 마이그레이션 순서, 시크릿 제외 등 |
| Shell-safe `git add` | zsh에서 붙여넣어도 깨지지 않게 `cd` + 경로 따옴표 강제 |
| Output template | 사용자가 복붙할 계획 형식 고정 |
| Hard rules | `git add`/`commit`/`push` 금지, “커밋해 줄까요?” CTA 금지 |

### 2.3 에이전트 워크플로 (요약)

1. 워크스페이스 git 루트를 우선. 형제 레포가 dirty해도, 이미 맥락에 있을 때만 언급.
2. 병렬로 수집: `git status -sb`, `git diff --stat HEAD`, `git diff --name-only HEAD`, `git log -8 --oneline`
3. 경로 + 가벼운 diff로 관심사별 묶음. hunk를 과도하게 읽지 않음.
4. 아래 템플릿으로 계획 출력 후 **중단**. 이 스킬 안에서 커밋 실행 제안하지 않음.

### 2.4 쪼개기 원칙 (스킬이 따르는 규칙)

- 커밋당 **하나의 관심사**. 메시지는 **why** 중심.
- 강하게 묶인 기능은 API+UI를 한 커밋(수직 슬라이스)으로.
- migration / schema / OpenAPI / 그걸 쓰는 기능은 같이 두거나 인접 순서로.
- 기능 / 버그픽스 / 순수 리팩터 / docs / chore는 분리.
- bisect·revert 친화 순서: 기반 → 기능 → polish/docs.
- 멀티 레포면 **레포별** 계획 + 의존 순서(예: module → host → BFF).
- `.env`, credentials 등 시크릿은 커밋 추천에서 **제외·경고**.
- 기존 `git log` 스타일(prefix, 언어, 톤)에 맞춤.

---

## 3. 어떻게 쓰나

### 3.1 설치 후 (일상 사용)

1. 아래 [설치 방법](#5-설치-방법)으로 스킬을 넣어 둡니다.
2. Cursor에서 **커밋을 쪼개고 싶은 프로젝트**를 연 뒤, 에이전트 채팅에 예를 들어:

   ```text
   Changes 커밋 어떻게 쪼개면 좋을까?
   ```
   ```text
   commit plan 짜줘. 커밋은 내가 할게.
   ```
   ```text
   커밋 쪼개기 추천만 해줘
   ```

3. 에이전트가 `commit-plan-only`를 고르면, `SKILL.md`대로 **계획만** 나옵니다.
4. 출력의 **Stage (you run)** 블록을 터미널에 붙여넣고, 메시지 초안을 보고 **직접** `git commit` 합니다.

스킬이 안 잡히면 description에 있는 트리거 문구를 그대로 쓰거나, 채팅에서 스킬을 명시적으로 지정하세요.

### 3.2 사용자가 하는 일 / 에이전트가 하는 일

| 주체 | 하는 일 |
|------|---------|
| 에이전트 | status/diff/log 읽기 → 단위·순서·메시지·quoted `git add` 블록 제안 |
| 사용자 | `cd` + `git add` 붙여넣기 → 메시지 확인 → `git commit` → 필요 시 push |

### 3.3 zsh에서 붙여넣을 때 (중요)

계획은 **zsh에 복붙**되는 것을 전제로 합니다.

- 각 Stage 블록은 **그 커밋이 속한 레포 루트로 `cd`** 한 뒤 `git add` (절대 경로 또는 `~/…`).
- 경로에 `[` `]` `?` `*` 공백 `!` 등이 있으면 **반드시 작은따옴표**.
- Cursor 워크스페이스 루트와 git 루트가 다를 수 있으므로, `cd` 없이 형제 레포 경로만 `git add` 하면 `pathspec did not match`가 납니다.

**Good**

```bash
cd ~/Desktop/example-module
git add \
  'packages/example/lib/src/domain/relative_time.dart'

cd ~/Desktop/example-app
git add \
  'apps/web/src/app/api/v1/members/[memberId]/children/route.ts' \
  'apps/web/src/hooks/useActiveBaby.tsx'
```

**Bad**

```bash
# 잘못된 cwd / 따옴표 없음 → zsh glob 실패
git add apps/web/src/app/api/v1/members/[memberId]/children/route.ts
```

파일 목록(Files) 줄의 백틱은 읽기용으로 괜찮고, **Stage 블록의 `git add`만** 따옴표를 강제합니다.

### 3.4 커밋까지 에이전트에게 맡기고 싶을 때

1. 이 스킬로 계획을 받습니다.
2. **새 메시지**로 “이 계획대로 커밋해줘”처럼 **명시적으로** 커밋을 요청합니다.
3. 그때부터는 이 스킬의 Hard rules 밖이며, 평소 committing 사용자 규칙을 따릅니다.

---

## 4. 에이전트가 내는 결과물

대략 이런 형태입니다 (`SKILL.md` Output template과 동일).

````markdown
## Commit plan — `<repo>`

### 1. `<type>: <short why>`
- **Files:** `path1`, `path2`, …
- **Why:** one line
- **Stage (you run):**
```bash
cd /absolute/or/~/path/to/<repo>
git add \
  'path/with/[brackets]/a.ts' \
  'path/without/special.tsx'
```

### Order
1 → 2 → …

### Notes
- leftovers, ambiguous files, risks
````

| 필드 | 의미 |
|------|------|
| `### N. …` | 커밋 단위 + 메시지 초안 (why) |
| **Files** | 이 커밋에 넣을 경로 (읽기용) |
| **Why** | 왜 이 묶음인지 한 줄 |
| **Stage** | 사용자가 실행할 `cd` + quoted `git add` |
| **Order** | 적용 순서 (기반 → 기능 → …) |
| **Notes** | 남은 파일, 애매한 묶음, 시크릿 등 |

### 함 / 안 함

| 함 | 안 함 |
|----|--------|
| `git status` / `diff` / `log` 읽기 | `git add` 실행 |
| 커밋 단위·순서·메시지 추천 | `git commit` / push / amend |
| 복사 가능한 `cd` + quoted `git add` | “이대로 커밋해 줄까요?”를 기본 CTA로 제시 |

---

## 5. 설치 방법

### A. skills CLI (권장)

GitHub에 이 레포를 올린 뒤 (레포 이름 예: `commit-plan-only`):

```bash
npx skills@latest add psm1st/commit-plan-only
```

단일 스킬 레포라 `--skill` 없이도 잡히는 경우가 많습니다. 실패하면:

```bash
npx skills@latest add psm1st/commit-plan-only --skill commit-plan-only
```

### B. Cursor: Remote GitHub / Team Marketplace

- Cursor Settings → Rules / Skills → GitHub에서 이 레포 추가
- 팀이면 Team Marketplace에 올리면 설치 UX가 더 단순합니다

### C. 수동 복사

이 폴더를 clone/다운로드한 뒤:

```bash
mkdir -p ~/.cursor/skills/commit-plan-only
cp SKILL.md ~/.cursor/skills/commit-plan-only/
```

개인용으로 `~/.cursor/skills/split-commits/`를 쓰고 있다면, **어느 쪽을 소스 오브 트루스**로 둘지 정한 뒤 복사해 드리프트를 막으세요.

---

## 6. 배포·유지보수

### 배포 전 체크리스트

1. GitHub에 `commit-plan-only` (또는 원하는 이름) 레포 생성 후 이 폴더를 연결
2. frontmatter `name: commit-plan-only` 유지
3. `SKILL.md`에 사내 전용 경로·비밀이 없는지 점검
4. 로컬 `split-commits` ↔ 이 레포 `SKILL.md` 동기화 정책 정하기
5. (선택) LICENSE (공개 시 MIT 등)

### GitHub에 올리는 예

```bash
cd ~/Desktop/commit-plan-only   # 또는 이 폴더 경로
git add .
git commit -m "$(cat <<'EOF'
Initial commit: commit-plan-only skill

EOF
)"
git remote add origin git@github.com:psm1st/commit-plan-only.git
git branch -M main
git push -u origin main
```

Cursor / GitHub UI의 “Publish Repository”로 올려도 됩니다.

### 에이전트가 잘 타게 하려면

- `description`에 트리거 문구를 넉넉히 (`commit plan`, `커밋 쪼개기`, …)
- Soft rules만 두지 말고 **Hard rules**로 add/commit 금지를 명시 (현재 `SKILL.md` 그대로)
- Output template을 고정해 복붙 품질을 안정화

---

커밋·푸시는 이 가이드를 보고 **직접** 하시면 됩니다.
