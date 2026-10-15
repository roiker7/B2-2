# `git reset` 완벽 정리 — `git reset --soft HEAD~1`부터 모든 옵션까지

> **한 줄 요약**
> `git reset --soft HEAD~1` = **"마지막 커밋만 취소하고, 그 커밋에 들어있던 변경 내용은 스테이징(add)된 상태 그대로 남겨둔다."**

---

## 목차

1. [먼저 알아야 할 Git의 3가지 영역](#1-먼저-알아야-할-git의-3가지-영역)
2. [`git reset --soft HEAD~1` 해부하기](#2-git-reset---soft-head1-해부하기)
3. [예시로 따라 해보기](#3-예시로-따라-해보기)
4. [`git reset`의 모드 옵션 총정리](#4-git-reset의-모드-옵션-총정리)
5. [커밋을 가리키는 방법 (`HEAD~1`, `HEAD^`, 해시 등)](#5-커밋을-가리키는-방법)
6. [기타 옵션 (`-q`, `-p`, `-N`, `--recurse-submodules` 등)](#6-기타-옵션)
7. [파일 단위 reset](#7-파일-단위-reset)
8. [실수했을 때 되돌리기 — `reflog`와 `ORIG_HEAD`](#8-실수했을-때-되돌리기--reflog와-orig_head)
9. [이미 push한 커밋이라면? — `reset` vs `revert`](#9-이미-push한-커밋이라면--reset-vs-revert)
10. [자주 쓰는 실전 시나리오](#10-자주-쓰는-실전-시나리오)
11. [비슷한 명령어와 비교](#11-비슷한-명령어와-비교)
12. [치트시트](#12-치트시트)

---

## 1. 먼저 알아야 할 Git의 3가지 영역

`git reset`을 이해하려면 Git이 파일을 관리하는 **세 공간**을 알아야 합니다.

| 영역 | 다른 이름 | 설명 | 이동 명령 |
|---|---|---|---|
| **Working Directory** | 작업 디렉터리, 워킹 트리 | 내가 실제로 파일을 수정하는 곳 | — |
| **Staging Area** | Index, 스테이징 영역 | 다음 커밋에 포함될 내용을 모아두는 곳 | `git add` |
| **Repository** | HEAD, 커밋 히스토리 | 확정된 커밋들이 저장되는 곳 | `git commit` |

```
 Working Directory  --git add-->  Staging Area  --git commit-->  Repository(HEAD)
    (파일 수정)                    (커밋 준비)                    (커밋 완료)
```

`git reset`은 이 흐름을 **거꾸로** 되돌리는 명령어이고,
**옵션(`--soft` / `--mixed` / `--hard`)에 따라 어디까지 되돌릴지**가 달라집니다.

### HEAD란?

- `HEAD`는 **"현재 내가 위치한 커밋"**을 가리키는 포인터입니다.
- 보통은 현재 브랜치(예: `main`)를 가리키고, 브랜치는 그 브랜치의 최신 커밋을 가리킵니다.
- `git reset`은 본질적으로 **"현재 브랜치(와 HEAD)가 가리키는 커밋을 다른 커밋으로 옮기는"** 명령입니다.

---

## 2. `git reset --soft HEAD~1` 해부하기

```bash
git reset --soft HEAD~1
```

| 부분 | 의미 |
|---|---|
| `git reset` | 현재 브랜치가 가리키는 커밋을 옮긴다 |
| `--soft` | **브랜치 포인터만** 옮기고, 스테이징 영역과 작업 디렉터리는 **건드리지 않는다** |
| `HEAD~1` | 현재 커밋(HEAD)의 **1단계 이전(부모) 커밋** |

### 실행 전후 그림

```
[실행 전]
A --- B --- C   ← main ← HEAD
            (C = 방금 한 커밋)

[git reset --soft HEAD~1 실행 후]
A --- B         ← main ← HEAD
       \
        C  (히스토리에서 사라짐, 하지만 reflog에는 남아 있음)

C에서 바뀐 내용 → 스테이징 영역에 "add된 상태"로 그대로 남아 있음
```

### 핵심 포인트

- ✅ 커밋 **C가 히스토리에서 빠진다** (`git log`에서 안 보임)
- ✅ C에서 수정했던 내용은 **사라지지 않는다**
- ✅ 그 내용은 **스테이징(`git add`)된 상태**로 남는다 → 바로 `git commit` 가능
- ✅ 작업 디렉터리의 파일 내용도 **그대로**
- ⚠️ 이미 원격 저장소에 push한 커밋이라면 주의 ([9장](#9-이미-push한-커밋이라면--reset-vs-revert) 참고)

### 언제 쓰나요?

- 커밋 메시지를 잘못 썼을 때
- 커밋에 파일을 빠뜨렸거나, 넣지 말아야 할 파일을 넣었을 때
- 커밋을 너무 잘게 나눠서 여러 개를 하나로 합치고 싶을 때 (squash)
- 커밋은 너무 일찍 했는데, 내용은 유지하고 싶을 때

---

## 3. 예시로 따라 해보기

아래 예시는 그대로 터미널에 따라 쳐보면서 공부할 수 있도록 구성했습니다.

### 3-1. 실습용 저장소 만들기

```bash
mkdir reset-practice && cd reset-practice
git init

echo "첫 번째 줄" > memo.txt
git add memo.txt
git commit -m "첫 커밋"

echo "두 번째 줄" >> memo.txt
git add memo.txt
git commit -m "두 번째 커밋"

echo "세 번째 줄" >> memo.txt
git add memo.txt
git commit -m "세번쨰 커밋 (오타!)"
```

```bash
$ git log --oneline
c3c3c3c (HEAD -> main) 세번쨰 커밋 (오타!)
b2b2b2b 두 번째 커밋
a1a1a1a 첫 커밋
```

> 해시값(`c3c3c3c` 등)은 예시이며 실제로는 다르게 나옵니다.

### 3-2. `--soft`로 마지막 커밋 취소하기

```bash
git reset --soft HEAD~1
```

```bash
$ git log --oneline
b2b2b2b (HEAD -> main) 두 번째 커밋
a1a1a1a 첫 커밋
```

마지막 커밋이 사라졌습니다. 그런데 변경 내용은?

```bash
$ git status
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   memo.txt
```

```bash
$ cat memo.txt
첫 번째 줄
두 번째 줄
세 번째 줄
```

👉 **"Changes to be committed"** = 스테이징된 상태로 남아 있습니다.
👉 파일 내용(`세 번째 줄`)도 그대로입니다.

### 3-3. 메시지를 고쳐서 다시 커밋하기

```bash
git commit -m "세 번째 커밋"
```

```bash
$ git log --oneline
d4d4d4d (HEAD -> main) 세 번째 커밋
b2b2b2b 두 번째 커밋
a1a1a1a 첫 커밋
```

### 3-4. 예시: 커밋에 파일을 빠뜨렸을 때

```bash
echo "print('hello')" > app.py
echo "API_KEY=secret" > config.txt
git add app.py
git commit -m "앱 추가"        # 앗, 다른 파일도 같이 올려야 했는데...

git reset --soft HEAD~1        # 커밋 취소 (app.py는 스테이징 유지)
echo "README 내용" > README.md
git add README.md               # 빠뜨린 파일 추가
git commit -m "앱과 README 추가"
```

### 3-5. 예시: 넣지 말아야 할 파일을 커밋했을 때

```bash
git add .
git commit -m "기능 추가"       # config.txt(비밀키)가 같이 들어감!

git reset --soft HEAD~1         # 커밋 취소, 전부 스테이징 상태로 복귀
git restore --staged config.txt # config.txt만 스테이징에서 제외
echo "config.txt" >> .gitignore # 다음부터 실수 방지
git add .gitignore
git commit -m "기능 추가"
```

> ⚠️ 비밀키를 이미 **push** 했다면, reset으로 지워도 이미 노출된 것으로 봐야 합니다. 키를 즉시 폐기/재발급하세요.

### 3-6. 예시: 여러 커밋을 하나로 합치기 (squash)

```bash
$ git log --oneline
e5e5e5e (HEAD -> main) 오타 수정
d4d4d4d 오타 수정 2
c3c3c3c 로그인 기능 구현
b2b2b2b 이전 작업
```

"로그인 기능 구현 + 오타 수정 2개"를 하나로 합치고 싶다면:

```bash
git reset --soft HEAD~3     # 최근 3개 커밋 취소, 변경 내용은 모두 스테이징
git commit -m "로그인 기능 구현"
```

```bash
$ git log --oneline
f6f6f6f (HEAD -> main) 로그인 기능 구현
b2b2b2b 이전 작업
```

---

## 4. `git reset`의 모드 옵션 총정리

`git reset`에는 **"어디까지 되돌릴지"**를 결정하는 모드 옵션이 5가지 있습니다.

```bash
git reset [--soft | --mixed | --hard | --merge | --keep] [<커밋>]
```

### 4-1. 한눈에 비교 (가장 중요!)

| 옵션 | HEAD(커밋) | 스테이징 영역 | 작업 디렉터리 | 변경 내용은? | 위험도 |
|---|:---:|:---:|:---:|---|:---:|
| `--soft` | ✅ 되돌림 | ❌ 유지 | ❌ 유지 | **스테이징된 상태**로 남음 | 🟢 안전 |
| `--mixed` (기본값) | ✅ 되돌림 | ✅ 되돌림 | ❌ 유지 | **수정됨(unstaged)** 상태로 남음 | 🟢 안전 |
| `--hard` | ✅ 되돌림 | ✅ 되돌림 | ✅ 되돌림 | **완전히 삭제됨** | 🔴 위험 |
| `--merge` | ✅ 되돌림 | ✅ 되돌림 | 일부 | 커밋 안 된 변경은 보존 시도 | 🟡 |
| `--keep` | ✅ 되돌림 | ✅ 되돌림 | 일부 | 로컬 변경 충돌 시 중단 | 🟡 |

```
                   HEAD      Staging     Working Dir
--soft     :      [이동]      그대로       그대로
--mixed    :      [이동]     [초기화]      그대로
--hard     :      [이동]     [초기화]     [초기화]   ← 수정 내용 증발!
```

### 4-2. `--soft` — 커밋만 취소

```bash
git reset --soft HEAD~1
```

- 브랜치 포인터만 이동
- 변경 내용 → **staged** (`Changes to be committed`)
- 사용처: 커밋 메시지 수정, 커밋 합치기, 빠뜨린 파일 추가

### 4-3. `--mixed` — 커밋 + add 취소 (기본값)

```bash
git reset --mixed HEAD~1
git reset HEAD~1            # 위와 동일 (옵션 생략 시 --mixed)
```

- 브랜치 포인터 이동 + 스테이징 영역 초기화
- 변경 내용 → **unstaged** (`Changes not staged for commit`)
- 사용처: 커밋을 취소하고 **어떤 파일을 add할지 처음부터 다시 고르고 싶을 때**

```bash
$ git reset HEAD~1
Unstaged changes after reset:
M       memo.txt

$ git status
Changes not staged for commit:
        modified:   memo.txt
```

> 💡 커밋 하나를 여러 개로 **쪼개고** 싶을 때 유용합니다.
> ```bash
> git reset HEAD~1
> git add login.py && git commit -m "로그인 기능"
> git add signup.py && git commit -m "회원가입 기능"
> ```

### 4-4. `--hard` — 전부 취소 (⚠️ 주의)

```bash
git reset --hard HEAD~1
```

- 브랜치 포인터 이동 + 스테이징 초기화 + **작업 디렉터리까지 초기화**
- 해당 커밋의 변경 내용이 **파일에서도 사라짐**
- **커밋하지 않은 수정사항도 모두 사라짐** → 이건 reflog로도 복구 불가!
- 사용처: "이 작업은 완전히 버리고 깨끗한 상태로 돌아가고 싶다"

```bash
git reset --hard HEAD        # 커밋은 그대로, 커밋 안 한 수정사항만 모두 버림
git reset --hard origin/main # 로컬을 원격 main과 완전히 똑같이 맞춤
```

> 🔴 `--hard` 실행 전 체크리스트
> 1. `git status`로 커밋 안 한 변경이 없는지 확인
> 2. 아깝다면 `git stash`로 임시 저장
> 3. 정말 버려도 되는지 한 번 더 생각

### 4-5. `--merge` — 머지 취소용

```bash
git reset --merge ORIG_HEAD
```

- 스테이징 영역은 초기화하고, 작업 디렉터리 중 **현재 커밋과 대상 커밋 간에 달라진 파일만** 갱신
- **커밋/스테이징하지 않은 로컬 변경은 최대한 보존**하고, 보존할 수 없으면 reset을 **중단**
- 사용처: 충돌 난 merge를 중단하거나, 방금 한 merge를 취소할 때
  (요즘은 `git merge --abort`를 더 많이 씁니다)

### 4-6. `--keep` — 로컬 변경을 지키면서 이동

```bash
git reset --keep HEAD~1
```

- `--hard`와 비슷하게 커밋과 파일을 되돌리지만,
- **커밋 안 한 로컬 변경이 있는 파일이 되돌릴 대상과 겹치면 reset을 중단**합니다
- 겹치지 않는 로컬 변경은 그대로 유지
- 사용처: "`--hard`처럼 커밋을 버리고 싶은데, 혹시 작업 중인 파일이 날아갈까 봐 불안할 때"

```bash
$ git reset --keep HEAD~1
error: Entry 'memo.txt' not uptodate. Cannot merge.
fatal: Could not reset index file to revision 'HEAD~1'.
# → 작업 중인 변경을 보호하기 위해 중단됨
```

### 4-7. 같은 상황에서 세 모드 비교 예시

상황: `memo.txt`에 "세 번째 줄"을 추가하고 커밋한 직후

| 명령 | `git log` | `git status` | `cat memo.txt` |
|---|---|---|---|
| `git reset --soft HEAD~1` | 커밋 사라짐 | `Changes to be committed: memo.txt` | 세 번째 줄 **있음** |
| `git reset --mixed HEAD~1` | 커밋 사라짐 | `Changes not staged: memo.txt` | 세 번째 줄 **있음** |
| `git reset --hard HEAD~1` | 커밋 사라짐 | `nothing to commit` | 세 번째 줄 **없음** |

---

## 5. 커밋을 가리키는 방법

`HEAD~1` 자리에 다양한 형태의 "커밋 지정자"를 넣을 수 있습니다.

| 표현 | 의미 | 예시 |
|---|---|---|
| `HEAD` | 현재 커밋 | `git reset --hard HEAD` (수정사항만 버림) |
| `HEAD~1` | 1단계 이전 커밋 (부모) | `git reset --soft HEAD~1` |
| `HEAD~` | `HEAD~1`과 동일 | `git reset --soft HEAD~` |
| `HEAD~3` | 3단계 이전 커밋 | `git reset --soft HEAD~3` |
| `HEAD^` | 첫 번째 부모 (`HEAD~1`과 동일) | `git reset --soft HEAD^` |
| `HEAD^^` | 부모의 부모 (`HEAD~2`와 동일) | `git reset HEAD^^` |
| `HEAD^2` | **두 번째 부모** (머지 커밋에서만 의미 있음) | 머지된 브랜치 쪽 부모 |
| `a1b2c3d` | 특정 커밋 해시 | `git reset --soft a1b2c3d` |
| `main`, `origin/main` | 브랜치가 가리키는 커밋 | `git reset --hard origin/main` |
| `v1.0` | 태그 | `git reset --hard v1.0` |
| `ORIG_HEAD` | reset/merge/rebase **직전** HEAD | `git reset ORIG_HEAD` |
| `HEAD@{2}` | reflog 기준 2번 전 HEAD 위치 | `git reset --hard HEAD@{2}` |

### `~`(틸드) vs `^`(캐럿) 차이

```
        D   ← 머지된 브랜치
         \
A --- B --- C --- M   ← HEAD (M은 머지 커밋)
```

- `HEAD~1` = `HEAD^` = `HEAD^1` = **C** (첫 번째 부모)
- `HEAD^2` = **D** (두 번째 부모)
- `HEAD~2` = **B** (첫 번째 부모를 두 번 따라감)

> 💡 일반적인 상황에서는 `HEAD~N`만 기억해도 충분합니다.

> ⚠️ **Windows CMD**에서는 `^`가 특수문자라서 `HEAD^`가 `HEAD`로 인식될 수 있습니다.
> `"HEAD^"`처럼 따옴표로 감싸거나 `HEAD~1`을 쓰세요.
> zsh에서도 `HEAD^`가 글롭 패턴으로 해석되어 에러가 날 수 있으니 `HEAD~1` 사용을 추천합니다.

### 커밋 지정자를 생략하면?

```bash
git reset          # = git reset --mixed HEAD  → 스테이징 전체 취소 (git add 취소)
git reset --soft   # = git reset --soft HEAD   → 아무 일도 일어나지 않음
git reset --hard   # = git reset --hard HEAD   → 커밋 안 한 변경 전부 삭제!
```

---

## 6. 기타 옵션

| 옵션 | 설명 | 예시 |
|---|---|---|
| `-q`, `--quiet` | 출력 메시지를 숨김 (에러만 출력) | `git reset -q --soft HEAD~1` |
| `--no-quiet` | `-q`를 끔 (설정으로 quiet가 켜져 있을 때) | `git reset --no-quiet HEAD~1` |
| `-p`, `--patch` | 변경 내용을 **덩어리(hunk) 단위로 골라서** 스테이징 취소 | `git reset -p` |
| `-N`, `--intent-to-add` | (`--mixed`와 함께) 삭제된 경로를 "추가 예정"으로 표시 | `git reset -N HEAD~1` |
| `--recurse-submodules` | 서브모듈의 작업 트리도 함께 reset | `git reset --hard --recurse-submodules HEAD` |
| `--no-refresh` | `--mixed` 후 인덱스 새로고침을 생략 (대형 저장소 속도용) | `git reset --no-refresh` |
| `--pathspec-from-file=<파일>` | 경로 목록을 파일에서 읽음 | `git reset --pathspec-from-file=list.txt` |
| `--pathspec-file-nul` | 위 파일의 구분자를 NUL 문자로 | 스크립트용 |
| `--` | 이후 인자는 모두 **파일 경로**로 해석 | `git reset -- main` (`main`이라는 파일) |

### `-p` (patch) 예시

```bash
$ git reset -p
diff --git a/memo.txt b/memo.txt
@@ -1,2 +1,3 @@
 첫 번째 줄
 두 번째 줄
+세 번째 줄
(1/1) Unstage this hunk [y,n,q,a,d,e,?]?
```

| 키 | 의미 |
|---|---|
| `y` | 이 덩어리 스테이징 취소 |
| `n` | 그대로 둠 |
| `q` | 종료 |
| `a` | 이 파일의 나머지 덩어리 모두 취소 |
| `d` | 이 파일의 나머지 덩어리 모두 그대로 둠 |
| `s` | 덩어리를 더 작게 쪼개기 |
| `e` | 직접 편집 |
| `?` | 도움말 |

---

## 7. 파일 단위 reset

커밋 지정자 뒤에 **파일 경로**를 주면, 브랜치 포인터는 움직이지 않고 **스테이징 영역의 해당 파일만** 되돌립니다.

```bash
git reset [<커밋>] [--] <파일경로>...
```

```bash
git add a.txt b.txt c.txt
git reset b.txt              # b.txt만 add 취소 (= git restore --staged b.txt)
git reset HEAD -- b.txt      # 위와 동일한 의미
git reset HEAD~1 -- a.txt    # a.txt의 스테이징 상태를 이전 커밋 버전으로 (작업 파일은 그대로)
```

> ⚠️ 파일 경로와 함께 `--soft`나 `--hard`는 **사용할 수 없습니다.**
> ```bash
> $ git reset --hard HEAD~1 a.txt
> fatal: Cannot do hard reset with paths.
> ```
> 파일 하나의 작업 내용을 되돌리려면 `git restore a.txt` 또는 `git checkout -- a.txt`를 쓰세요.

---

## 8. 실수했을 때 되돌리기 — `reflog`와 `ORIG_HEAD`

`reset`으로 커밋을 "지워도" Git은 실제로 바로 삭제하지 않습니다.
**HEAD가 움직인 기록(reflog)**이 남아 있어서 대부분 복구할 수 있습니다.

### 8-1. `ORIG_HEAD`로 즉시 되돌리기

`reset`, `merge`, `rebase`를 하면 Git은 직전 위치를 `ORIG_HEAD`에 저장합니다.

```bash
git reset --soft HEAD~1    # 앗, 취소하면 안 되는 거였다!
git reset ORIG_HEAD        # 바로 직전 상태로 복구
```

> 💡 `--soft`로 취소한 직후라면 `git reset --soft ORIG_HEAD`로 원래대로 돌아갈 수 있습니다.

### 8-2. `git reflog`로 찾아서 되돌리기

```bash
$ git reflog
b2b2b2b (HEAD -> main) HEAD@{0}: reset: moving to HEAD~1
c3c3c3c HEAD@{1}: commit: 세번쨰 커밋 (오타!)
b2b2b2b HEAD@{2}: commit: 두 번째 커밋
a1a1a1a HEAD@{3}: commit (initial): 첫 커밋
```

```bash
git reset --hard c3c3c3c   # 해시로 복구
git reset --hard HEAD@{1}  # reflog 번호로 복구
```

### 8-3. 복구 가능 / 불가능 정리

| 상황 | 복구 가능? | 방법 |
|---|:---:|---|
| `--soft`/`--mixed`/`--hard`로 **커밋**을 취소함 | ✅ | `reflog` 또는 `ORIG_HEAD` |
| `--hard`로 **커밋 안 한** 수정사항이 날아감 | ❌ | 거의 불가능 (에디터 로컬 히스토리 정도) |
| `--hard`로 **add만 하고 커밋 안 한** 파일이 날아감 | △ | `git fsck --lost-found`로 blob을 찾을 수 있는 경우 있음 |

> reflog 기록은 기본적으로 약 **90일**(도달 불가능한 커밋은 30일) 보관되다가 `git gc` 시 정리됩니다.

---

## 9. 이미 push한 커밋이라면? — `reset` vs `revert`

`reset`은 **히스토리를 다시 쓰는** 명령입니다.
이미 원격(GitHub)에 push한 커밋을 reset하면 로컬과 원격의 히스토리가 달라집니다.

```bash
git reset --soft HEAD~1
git commit -m "수정된 커밋"
git push
# ! [rejected]  main -> main (non-fast-forward)
```

### 방법 1: 강제 push (혼자 쓰는 브랜치일 때만)

```bash
git push --force-with-lease
```

- `--force`보다 `--force-with-lease`가 안전합니다
  (그 사이 다른 사람이 push한 내용이 있으면 거부해줌)
- ⚠️ **팀원과 공유하는 브랜치(main 등)에서는 절대 비추천** — 다른 사람의 히스토리가 꼬입니다

### 방법 2: `git revert` (공유 브랜치에서 권장)

```bash
git revert HEAD
```

- 커밋을 지우는 대신, **"그 커밋을 되돌리는 새 커밋"**을 만듭니다
- 히스토리가 보존되므로 협업에서 안전합니다

| 구분 | `git reset` | `git revert` |
|---|---|---|
| 동작 | 커밋을 히스토리에서 제거 | 반대 내용의 새 커밋 추가 |
| 히스토리 | 변경됨 (다시 씀) | 보존됨 |
| push 후 사용 | ⚠️ force push 필요 | ✅ 일반 push 가능 |
| 추천 상황 | 아직 push 안 한 로컬 커밋 | 이미 공유된 커밋 |

---

## 10. 자주 쓰는 실전 시나리오

| 하고 싶은 것 | 명령어 |
|---|---|
| 마지막 커밋 취소, 변경은 **staged**로 유지 | `git reset --soft HEAD~1` |
| 마지막 커밋 취소, 변경은 **unstaged**로 유지 | `git reset HEAD~1` |
| 마지막 커밋 **완전히** 삭제 | `git reset --hard HEAD~1` |
| 최근 N개 커밋을 하나로 합치기 | `git reset --soft HEAD~N` → `git commit` |
| 커밋 하나를 여러 개로 쪼개기 | `git reset HEAD~1` → 나눠서 `add`/`commit` |
| `git add` 전체 취소 | `git reset` |
| 특정 파일만 `git add` 취소 | `git reset <파일>` 또는 `git restore --staged <파일>` |
| 커밋 안 한 수정사항 전부 버리기 | `git reset --hard HEAD` |
| 로컬을 원격과 똑같이 맞추기 | `git fetch` → `git reset --hard origin/main` |
| 방금 한 reset 되돌리기 | `git reset ORIG_HEAD` (또는 `git reflog`로 찾기) |
| 방금 한 merge 취소 | `git reset --merge ORIG_HEAD` |
| 실수로 main에 커밋함 → 새 브랜치로 옮기기 | 아래 참고 |

### 시나리오: 실수로 main에 커밋했을 때

```bash
# main에서 작업하다가 커밋까지 해버림
git branch feature/login     # 현재 커밋을 가리키는 새 브랜치 생성
git reset --hard HEAD~1      # main은 이전 커밋으로 되돌림
git switch feature/login     # 새 브랜치로 이동해서 작업 계속
```

> 커밋이 새 브랜치에 남아 있으므로 `--hard`를 써도 안전합니다.

---

## 11. 비슷한 명령어와 비교

### `git reset --soft HEAD~1` vs `git commit --amend`

| 구분 | `git reset --soft HEAD~1` → `git commit` | `git commit --amend` |
|---|---|---|
| 목적 | 마지막 커밋 다시 만들기 | 마지막 커밋 수정 |
| 결과 | 거의 동일 | 거의 동일 |
| 편의성 | 두 단계 | 한 단계 |
| 여러 커밋 | `HEAD~N`으로 가능 | 마지막 1개만 |

```bash
# 메시지만 바꾸기
git commit --amend -m "새 메시지"

# 빠뜨린 파일 추가 (메시지 유지)
git add 빠진파일.txt
git commit --amend --no-edit
```

### `git reset` vs `git restore` vs `git checkout`

| 명령 | 주 용도 | 브랜치 이동 |
|---|---|---|
| `git reset` | 커밋 되돌리기, 스테이징 취소 | 브랜치 포인터를 옮김 |
| `git restore` | 파일 내용/스테이징 복원 (Git 2.23+) | 안 옮김 |
| `git restore --staged <파일>` | 스테이징 취소 (= `git reset <파일>`) | 안 옮김 |
| `git checkout` / `git switch` | 다른 브랜치/커밋으로 이동 | HEAD만 이동 (브랜치는 그대로) |

> 💡 `git reset`은 **브랜치 자체**를 옮기고, `git checkout`/`git switch`는 **HEAD가 다른 곳을 보도록** 할 뿐 브랜치는 건드리지 않습니다.

### `git reset --soft` vs `git stash`

| 구분 | `git reset --soft HEAD~1` | `git stash` |
|---|---|---|
| 대상 | **커밋된** 내용 | **커밋 안 한** 내용 |
| 결과 | 커밋을 풀어서 staged로 | 변경을 임시 보관함에 저장 |

---

## 12. 치트시트

```bash
# ───────── 커밋 취소 ─────────
git reset --soft  HEAD~1     # 커밋 취소, 변경은 staged     🟢
git reset --mixed HEAD~1     # 커밋 취소, 변경은 unstaged   🟢 (= git reset HEAD~1)
git reset --hard  HEAD~1     # 커밋 + 변경 모두 삭제        🔴

# ───────── 여러 개 ─────────
git reset --soft HEAD~3      # 최근 3개 커밋 취소 (합치기용)

# ───────── 특정 커밋으로 ─────────
git reset --soft a1b2c3d     # 해당 커밋으로 이동
git reset --hard origin/main # 원격과 동일하게

# ───────── 스테이징 취소 ─────────
git reset                    # 전체 add 취소
git reset 파일명              # 특정 파일 add 취소
git reset -p                 # 덩어리 단위로 골라서 취소

# ───────── 복구 ─────────
git reset ORIG_HEAD          # 직전 reset 되돌리기
git reflog                   # HEAD 이동 기록 보기
git reset --hard HEAD@{1}    # reflog 위치로 복구

# ───────── push한 커밋 ─────────
git revert HEAD              # 공유 브랜치에서는 이걸로
git push --force-with-lease  # 개인 브랜치에서 reset 후 push
```

### 기억하기 쉬운 비유 📦

> 택배 상자(커밋)를 이미 포장해서 봉인했다고 생각해 보세요.
>
> - `--soft` : **봉인 테이프만 뜯는다** → 물건은 상자 안에 그대로 (staged)
> - `--mixed` : **상자에서 물건을 꺼내 책상에 놓는다** → 물건은 있지만 상자 밖 (unstaged)
> - `--hard` : **상자와 물건을 통째로 버린다** → 아무것도 안 남음 (삭제)

---

## 참고 자료

- [Git 공식 문서 — git-reset](https://git-scm.com/docs/git-reset)
- [Pro Git 책 (한국어) — Reset 명확히 알고 가기](https://git-scm.com/book/ko/v2/Git-%EB%8F%84%EA%B5%AC-Reset-%EB%AA%85%ED%99%95%ED%9E%88-%EC%95%8C%EA%B3%A0-%EA%B0%80%EA%B8%B0)
- [Git 공식 문서 — git-reflog](https://git-scm.com/docs/git-reflog)
- [Git 공식 문서 — git-revert](https://git-scm.com/docs/git-revert)
