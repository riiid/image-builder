# Upstream Sync Guide

이 fork (`riiid/image-builder`) 의 **upstream (`kubernetes-sigs/image-builder`) 최신 변경을 가져오는 절차**.
분기별 또는 K8s major release 시 수행 권장.

---

## 사전 개념

### Fork 의 본질

> Git 자체에는 "fork" 명령이 없습니다. **Fork 는 GitHub 의 server-side clone + 부모 관계 메타데이터**.

```
kubernetes-sigs/image-builder (upstream, 원본)
        │
        └─ GitHub server-side clone + metadata
                ▼
       riiid/image-builder (우리 fork — push 권한 보유)
```

### Git remote 2개의 역할

한 로컬 repo 에 두 개의 remote 등록:

| Remote 이름 | 가리키는 곳 | 권한 | 용도 |
|------------|----------|------|------|
| `origin` | `github.com/riiid/image-builder` | fetch + push | 우리 fork — push 대상 |
| `upstream` | `github.com/kubernetes-sigs/image-builder` | fetch only | 원본 — 최신 변경 가져오기 |

> `origin` / `upstream` 은 관습적 이름. 다른 이름도 가능하지만 표준 사용.

---

## 초기 설정 (최초 1회)

```bash
cd /path/to/image-builder

# upstream remote 등록
git remote add upstream https://github.com/kubernetes-sigs/image-builder.git

# 확인
git remote -v
# origin     git@github.com:riiid/image-builder.git (fetch)
# origin     git@github.com:riiid/image-builder.git (push)
# upstream   https://github.com/kubernetes-sigs/image-builder.git (fetch)
# upstream   https://github.com/kubernetes-sigs/image-builder.git (push)  ← 우리 권한 없으므로 사용 안 함
```

---

## Sync 절차 — CLI 5단계

### Step 1: upstream 최신 fetch

```bash
git fetch upstream
```

- upstream 의 모든 branch + tag 를 로컬에 가져옴
- working tree 영향 없음 (단순 다운로드)
- `upstream/main` reference 가 로컬에 생김

### Step 2: 변경 사항 확인 (sync 전 점검)

```bash
# upstream 에 우리보다 많은 commit 보기
git log --oneline main..upstream/main | head -20

# 우리만 있는 commit (보존 대상)
git log --oneline upstream/main..main

# 변경된 파일 확인
git diff main..upstream/main --stat
```

> ⚠️ `packer/proxmox/packer.json.tmpl` 이 변경됐는지 특히 주목.
> 우리 fix 4종이 있는 파일이라 conflict 가능성 높음.

### Step 3: 우리 main 으로 이동

```bash
git checkout main
```

### Step 4: 통합 — merge 또는 rebase

**Rebase 권장** (linear history, 우리 fix 가 1개라 깔끔):

```bash
git rebase upstream/main
```

또는 Merge (history 보존):

```bash
git merge upstream/main
```

#### Conflict 발생 시 (자세히는 [Conflict 해결](#conflict-해결) 섹션)

```
CONFLICT (content): Merge conflict in images/capi/packer/proxmox/packer.json.tmpl
```

→ 파일 수동 편집 → `git add` → `git rebase --continue` (rebase) 또는 `git commit` (merge)

### Step 5: 결과를 fork 에 push

```bash
# rebase 후에는 force push 필요 (history 재작성됨)
git push --force-with-lease origin main

# merge 후에는 일반 push
git push origin main
```

> ⚠️ **`--force-with-lease`** 사용 — 다른 사람이 동시에 push 했으면 실패 (안전).
> `--force` 단독은 위험 (남의 commit 덮어쓸 수 있음).

---

## Sync 절차 — GitHub UI (간단한 경우)

Conflict 가 없을 거라 예상되는 경우 한 클릭 방법:

```
https://github.com/riiid/image-builder
  → "Sync fork" 버튼 클릭
  → upstream 의 새 commit fetch + auto merge
  → 성공: "This branch is up to date with kubernetes-sigs:main"
  → 실패: "manual sync needed" → CLI 절차로 진행
```

**장점**: 한 클릭, 빠름
**단점**: conflict 시 결국 로컬에서 해결해야 함

---

## Merge vs Rebase 선택

### 비교 예시

**전제**:
- upstream 에 새 commit 3개 추가 (D, E, F)
- 우리 fork 에 fix commit 1개 (X)

**Merge 결과** (history 보존):

```
A → B → C → D → E → F (upstream)
            ↓           ↘
            └─ X    →    M (merge commit, 우리 main)
```

**Rebase 결과** (linear):

```
A → B → C → D → E → F → X' (X 가 새 SHA 로 재배치, 우리 main)
```

### 선택 기준

| 상황 | 추천 |
|------|------|
| 우리 fix 가 적고 깔끔한 linear history 원함 | **rebase** ✅ (우리 케이스) |
| 우리 fix 가 PR 머지 commit 등 history 보존 가치 있음 | merge |
| fork 에 기여자가 여럿 | merge |
| 학습용 personal fork | rebase (단순) |

→ **`riiid/image-builder` 는 rebase 권장**.

---

## Conflict 해결

### 발생 시나리오 (실제 예상)

upstream 이 `packer.json.tmpl` 의 builder block 수정 + 우리 fix 도 같은 블록 변경:

```json
"vm_id": "{{user `vmid`}}",
<<<<<<< HEAD (우리)
"qemu_agent": true
=======
"cores": "{{user `cores`}}"
>>>>>>> upstream/main
```

### 해결 절차

1. **파일 수동 편집** — 두 변경 모두 보존 (우리 fix + upstream 신규):

```json
"vm_id": "{{user `vmid`}}",
"qemu_agent": true,
"cores": "{{user `cores`}}"
```

2. **conflict marker 모두 제거** 확인:

```bash
# marker 검색 — 결과 없어야 함
grep -rE '<<<<<<<|=======|>>>>>>>' .
```

3. **continue rebase (또는 merge commit)**:

```bash
git add images/capi/packer/proxmox/packer.json.tmpl

# rebase 의 경우
git rebase --continue

# merge 의 경우
git commit   # editor 열림 → 메시지 확인 후 저장
```

### Rebase 중단 (rollback)

뭔가 잘못됐을 때:

```bash
git rebase --abort
```

→ rebase 시작 전 상태로 완전 복귀. 안전.

---

## 검증 — Sync 후 확인 사항

### Step 1: 우리 fix 4종 여전히 적용됐는지

```bash
cd images/capi/packer/proxmox
grep 'qemu_agent.*true' packer.json.tmpl       # Fix #1
grep 'needrestart' packer.json.tmpl            # Fix #2
grep 'nohup bash' packer.json.tmpl             # Fix #3
grep 'pause_before.*"90s"' packer.json.tmpl    # Fix #4
```

→ 4개 모두 결과 있어야 함.

### Step 2: upstream 의 새 기능 도입 확인

```bash
# upstream 의 새 commit log 보기
git log --oneline -10
```

### Step 3: build 동작 검증 (필요 시)

다음 K8s 빌드 시점에 실제 검증. sync 만 했다고 바로 빌드 안 해도 OK.

---

## 분기별 sync 운영 권장

K8s release cycle 이 약 4개월. **분기마다 한 번** sync 권장:

| 시점 | 이유 |
|------|------|
| K8s minor 업그레이드 전 (1.35, 1.36 등) | 빌드 직전 upstream 최신 적용 |
| 분기별 (3개월) | 누적 conflict 방지. 작은 sync 자주 |
| 보안 이슈 발생 시 | upstream 의 보안 fix 즉시 적용 |

---

## TODO — 자동화

### GitHub Actions 로 분기별 sync 자동화

향후 검토:

```yaml
# .github/workflows/sync-upstream.yml
name: Sync with upstream
on:
  schedule:
    - cron: '0 0 1 */3 *'   # 분기별 1일 자정
  workflow_dispatch:         # 수동 트리거

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          git remote add upstream https://github.com/kubernetes-sigs/image-builder.git
          git fetch upstream
          git checkout main
          git rebase upstream/main || exit 1   # conflict 시 실패 — Slack 알림
          git push origin main --force-with-lease
```

conflict 발생 시 Slack 알림 + 수동 해결 권장.

---

## 빠른 참조

```bash
# 처음 한 번
git remote add upstream https://github.com/kubernetes-sigs/image-builder.git

# 매번 sync (CLI)
git fetch upstream
git checkout main
git rebase upstream/main
git push --force-with-lease origin main

# 또는 GitHub UI 의 "Sync fork" 버튼 (conflict 없을 때)
```

---

## 관련 문서

- [`HISTORY.md`](./HISTORY.md) — fork 의 4개 fix 메커니즘 및 변경 이력
- [GitHub Fork 공식 문서](https://docs.github.com/en/get-started/quickstart/fork-a-repo)
- [Git rebase 공식 문서](https://git-scm.com/docs/git-rebase)
