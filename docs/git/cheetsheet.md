# 🚀 DevOps Git Cheat Sheet (Oh-My-Zsh Edition)

## 1\. 🌟 The One Pick (가장 추천하는 명령어)

팀의 커밋 히스토리를 깔끔하게 유지하는 비결입니다.

| Alias | 실제 명령어 | 효과 |
| :--- | :--- | :--- |
| **`gup`** | `git pull --rebase` | 일반 `pull`과 달리, **불필요한 Merge 커밋을 만들지 않고** 내 커밋을 최신 코드 뒤에 깔끔하게 이어 붙입니다. |

## 2\. 필수 설정 & 트러블슈팅

환경 세팅과 `.gitignore` 문제 해결용입니다.

| 상황 | 명령어 / 설명 |
| :--- | :--- |
| **유저 설정** | `git config --global user.name "내이름"`<br>`git config --global user.email "이메일"` |
| **.gitignore<br>미적용 해결** | **(캐시 삭제 후 재커밋)**<br>1. `git rm -r --cached .`<br>2. `git add .`<br>3. `git commit -m "Fix .gitignore"` |

## 3\. 실전 핵심 Alias (단축키)

자주 쓰는 순서대로 정리했습니다.

### A. 작업 시작 & 저장 (Basic)

| Alias | 실제 명령어 | 설명 |
| :--- | :--- | :--- |
| **`gst`** | `git status` | **[습관]** 현재 파일 상태 확인 (수시로 입력) |
| **`gaa`** | `git add --all` | 변경된 **모든** 파일을 스테이징 |
| **`gcmsg`** | `git commit -m` | 메시지와 함께 커밋 (`gcmsg "메시지"`) |
| **`gc!`** | `git commit --amend` | **[수정]** 방금 올린 커밋 내용/메시지 고치기 |
| **`gd`** | `git diff` | 코드 변경점 확인 (`add` 하기 전) |

### B. 업로드 & 동기화 (Sync)

| Alias | 실제 명령어 | 설명 |
| :--- | :--- | :--- |
| **`gp`** | `git push` | 원격 저장소로 업로드 |
| **`gl`** | `git pull` | 원격 저장소 내용 가져오기 (일반) |
| **`gup`** | `git pull --rebase` | **[⭐추천]** 깔끔하게 당겨오기 (Merge bubble 방지) |

### C. 브랜치 관리 (Branch)

| Alias | 실제 명령어 | 설명 |
| :--- | :--- | :--- |
| **`gsw`** | `git switch` | 브랜치 이동 (`gsw [브랜치명]`) |
| **`gswc`** | `git switch -c` | 새 브랜치 생성 후 이동 (`gswc [새브랜치명]`) |
| **`gb`** | `git branch` | 로컬 브랜치 목록 보기 |
| **`gbD`** | `git branch -D` | 브랜치 강제 삭제 |
| **`gm`** | `git merge` | 브랜치 합치기 |

### D. 임시 저장 (Stash)

작업 중 급하게 다른 일을 해야 할 때 사용합니다.

| Alias | 실제 명령어 | 설명 |
| :--- | :--- | :--- |
| **`gsta`** | `git stash push` | 작업 코드 임시 보관함에 숨기기 |
| **`gstp`** | `git stash pop` | 숨겨둔 코드 다시 꺼내오기 |
| **`gstl`** | `git stash list` | 보관된 목록 확인 |

### E. 되돌리기 (Undo/Reset)

⚠️ 주의해서 사용하세요.

| Alias | 실제 명령어 | 설명 |
| :--- | :--- | :--- |
| **`grh`** | `git reset --hard` | 특정 시점으로 **완전 초기화** (복구 불가) |
| **`grs`** | `git reset --soft` | 커밋만 취소하고, **파일 변경분은 남김** |

### F. 이력 조회 (Log)

| Alias | 실제 명령어 | 설명 |
| :--- | :--- | :--- |
| **`glo`** | `git log --oneline --graph` | 커밋 히스토리를 그래프로 깔끔하게 보기 |
