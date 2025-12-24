## 1. 목적
- **(1) Global 설정은 기본 GitHub 계정만 사용**
- **(2) `~/Document/dev/otherProject/` 아래 repo에서만 a 계정 자동 전환**
- **(3) Global 설정(`~/.gitconfig`)에 a 계정 정보가 전혀 없도록 완전 격리**

> 이번 버전의 전제(중요): **“계정 전환”을 Host alias가 아니라 “서로 다른 호스트”로 분리**해서 해결  
> - 기본 계정: `github.com`  
> - a 계정: `git.{your_company_name}.com` (예: GitHub Enterprise / 사내 Git 서버)

---

## 2. 전체 구조

### 파일/역할 맵
- `~/.ssh/config` (**인증 분리**)  
  - `Host github.com` → 기본 계정용 키 사용  
  - `Host git.{your_company_name}.com` → a 계정용 키 사용  
  - 이렇게 **host를 실제 도메인으로 쓰는 이유**: 5단계에서 remote를 `git@github.com:OWNER/REPO.git` 같은 **표준 형태**로 쓰기 위함
- `~/.gitconfig` (**Global Git 설정**)  
  - 기본 계정 `user.name/email`만 존재  
  - `includeIf "gitdir:~/Document/dev/otherProject/" ...` 규칙만 존재 (**a 정보 없음**)
- `~/.gitconfig-otherProject-a` (**a 계정 전용 Git 설정**)  
  - a 계정 `user.name/email`만 존재 (커밋 작성자용)

### 자동 전환이 일어나는 조건(2개 축)
- **커밋 작성자 전환**: repo가 `~/Document/dev/otherProject/` 아래면 `includeIf`로 a 작성자 적용
- **인증(푸시 권한) 전환**: remote의 host가  
  - `github.com`이면 기본 키로 인증  
  - `git.{your_company_name}.com`이면 a 키로 인증

---

## 3. 적용

### 1단계) 계정별 SSH 키 생성 + (선택) ssh-agent 등록 + 확인(ssh -T)

#### 1-1) 키 생성
```bash
# 기본 계정 키 (github.com 용)
ssh-keygen -t ed25519 -C "default_email@example.com" -f ~/.ssh/id_ed25519_gh_default

# a 계정 키 (git.{your_company_name}.com 용)
ssh-keygen -t ed25519 -C "a_email@example.com" -f ~/.ssh/id_ed25519_gh_a
```

#### 1-2) 서버에 public key 등록
```bash
cat ~/.ssh/id_ed25519_gh_default.pub
cat ~/.ssh/id_ed25519_gh_a.pub
```
- `github.com`의 기본 계정 SSH Keys에 `...default.pub` 등록  
- `git.{your_company_name}.com`의 a 계정(또는 사내 계정) SSH Keys에 `...a.pub` 등록

#### 1-3) (선택) `ssh-add`로 ssh-agent에 키 등록
```bash
ssh-add ~/.ssh/id_ed25519_gh_default
ssh-add ~/.ssh/id_ed25519_gh_a
```
등록 확인:
```bash
ssh-add -l
```

#### 1-4) (2단계 설정 후) `ssh -T`로 인증 확인
```bash
ssh -T git@github.com
ssh -T git@git.{your_company_name}.com
```

#### 이 단계에서 정리할 개념
✅ `ssh-add`는 “키 파일을 agent에 로드하는 명령”
- `ssh-agent`가 키를 들고 **서명 작업을 대신**해줘서 편함(패스프레이즈 입력 최소화)

✅ `ssh -T`는 “SSH 인증만 테스트”
- 실제로 repo를 클론/푸시하기 전에 **키가 서버에 잘 매핑되는지** 확인할 때 씀

✅ `-C` 이메일은 “증명”이 아니라 **라벨(comment)**
- 키 구분용 메모일 뿐, 이메일 소유를 검증하지 않음

✅ `ed25519`는 키 알고리즘 타입  
- 현대 SSH에서 널리 쓰는 안전하고 효율적인 키 타입

---

### 2단계) `~/.ssh/config`에 Host별 키 지정 + `Permission denied (publickey)` 해결 가이드

`~/.ssh/config`에 추가(예시):
```sshconfig
Host github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_gh_default
  IdentitiesOnly yes

Host git.{your_company_name}.com
  User git
  IdentityFile ~/.ssh/id_ed25519_gh_a
  IdentitiesOnly yes
```

> 여기서 **Host를 `github.com`, `git.{your_company_name}.com`으로 하는 이유**:  
> 5단계에서 remote를 `git@github.com:OWNER/REPO.git`처럼 **표준 URL 그대로** 쓰면, SSH가 `Host github.com` 블록을 자동 적용하게 만들기 위함.

#### 2-1) `ssh -T`로 바로 점검
```bash
ssh -T git@github.com
ssh -T git@git.{your_company_name}.com
```

#### 2-2) 만약 `Permission denied (publickey)`가 뜨면 (해결 순서)
아래를 **위에서부터 순서대로** 보면 대부분 잡혀.

1) **어떤 키를 실제로 내미는지 로그로 확인**
```bash
ssh -vT git@github.com
# 또는
ssh -vT git@git.{your_company_name}.com
```
- 로그에 `Offering public key ...`가 보임  
- 여기 나온 키가 **내가 의도한 IdentityFile인지** 확인

2) **`~/.ssh/config`가 제대로 적용되는지 확인**
- 오타/들여쓰기/대소문자 문제 확인 (특히 `IdentityFile` 경로)
- `IdentityFile` 파일이 실제 존재하는지:
```bash
ls -l ~/.ssh/id_ed25519_gh_default ~/.ssh/id_ed25519_gh_a
```

3) **public key를 서버 계정에 등록했는지 재확인**
- GitHub / 사내 Git의 “SSH keys”에 **`.pub` 내용**이 정확히 등록돼 있어야 함  
- (가장 흔한 원인) 다른 키를 등록했거나, 복사 중 줄바꿈/공백이 깨진 경우

4) **키 파일 권한(퍼미션) 확인**
- SSH는 키 파일 권한이 너무 열려 있으면 보안을 이유로 무시할 수 있어
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519_gh_default ~/.ssh/id_ed25519_gh_a
chmod 644 ~/.ssh/id_ed25519_gh_default.pub ~/.ssh/id_ed25519_gh_a.pub
```

5) **agent 관련 이슈(키가 없거나 너무 많거나)**
- 등록된 키 확인:
```bash
ssh-add -l
```
- 필요하면 한 번 비우고 필요한 키만 다시 올려보기:
```bash
ssh-add -D
ssh-add ~/.ssh/id_ed25519_gh_default
ssh-add ~/.ssh/id_ed25519_gh_a
```
- 단, 이 구성은 `IdentitiesOnly yes`가 있어서 agent에 키가 많아도 보통 안전함

6) **서버가 `git` 유저로 SSH를 받는지 확인**
- 어떤 서버는 SSH user가 `git`이 아닐 수도 있어(사내 정책)  
  그 경우 `User git` 줄이 서버 정책과 안 맞으면 실패할 수 있음  
  → 사내 가이드에 맞는 user가 있는지 확인

#### 이 단계에서 정리할 개념
✅ `Host`는 “SSH 설정 매칭 키(대상 이름)”  
- `ssh -T git@github.com`을 실행하면, SSH는 `Host github.com` 블록을 찾아 적용

✅ `IdentitiesOnly yes`는 “지정한 키만 써라”
- agent나 기본키를 무시하고 **IdentityFile만 사용** → 계정 섞임 방지

✅ `Permission denied (publickey)`는 “서버가 제출된 키를 인정 못했다”는 뜻
- 거의 항상 원인은: **잘못된 키 사용 / 서버에 public key 미등록 / 퍼미션 문제 / user 정책 불일치**

---

### 3단계) Global Git 설정: 기본 계정만 + includeIf 규칙만 (a 정보 0)

Global 기본 작성자 설정:
```bash
git config --global user.name  "DEFAULT_NAME"
git config --global user.email "default_email@example.com"
```

`~/.gitconfig`에 규칙 추가:
```ini
[includeIf "gitdir:~/Document/dev/otherProject/"]
  path = ~/.gitconfig-otherProject-a
```

#### 이 단계에서 정리할 개념
✅ `includeIf "gitdir:.../"`는 “폴더 조건부 설정 로드”
- repo가 해당 경로 아래면 **추가 설정 파일을 자동 include**

✅ Global에는 a 정보가 없어야 “완전 격리”
- 글로벌 파일에는 **a name/email/키 정보 없음**
- 오직 “조건부 include 규칙”만 존재

---

### 4단계) a 계정 전용 Git 설정 파일 만들기 (커밋 작성자만)

`~/.gitconfig-otherProject-a` 생성:
```ini
[user]
  name = A_NAME
  email = a_email@example.com
```

#### 이 단계에서 정리할 개념
✅ Git의 `user.name/email`은 “GitHub 로그인”이 아니라 “커밋 작성자”
- 인증(푸시 권한)은 SSH 키/서버가 결정
- 커밋에 찍히는 작성자 표시는 Git 설정이 결정

---

### 5단계) clone / remote URL 규칙 + “git@github.com:...로 클론하면 문제 없나?” 추가

#### 5-1) 표준 remote URL을 이렇게 쓴다
- GitHub(기본 계정):
  - `git@github.com:OWNER/REPO.git`
- 사내 Git(a 계정):
  - `git@git.{your_company_name}.com:OWNER/REPO.git`  
  (사내 서버의 경로 규칙은 조직마다 다를 수 있음)

#### 5-2) “`git@github.com:OWNER/REPO.git`로 클론하면 문제 없는가?”
✅ **문제 없음** — 단, 의미는 명확해:
- 그 URL로 클론하면 SSH는 `Host github.com` 설정을 적용하고
- 그 결과 **기본 계정 키로 인증**해서 clone/push하게 됨

👉 따라서:
- **otherProject가 ‘사내 Git 서버(repo host가 git.{your_company_name}.com)’에 있어야 a로 자동 전환**이 자연스럽게 성립  
- 만약 otherProject repo가 사실 GitHub(`github.com`)에 있는데 “a라는 다른 GitHub 계정”으로 푸시하고 싶다면, **이 구성(Host를 도메인만으로 분리)만으로는 불가능**하고, 그때는 다시:
  - Host alias(`github-a` 같은 별명) 방식 또는
  - repo별 `core.sshCommand` 방식  
  중 하나가 필요해

#### 5-3) 이미 클론된 repo의 remote 확인/변경
현재 remote 확인:
```bash
git remote -v
```

remote 변경:
```bash
# GitHub로 보내고 싶다면
git remote set-url origin git@github.com:OWNER/REPO.git

# 사내 Git 서버로 보내고 싶다면
git remote set-url origin git@git.{your_company_name}.com:OWNER/REPO.git
```

#### 이 단계에서 정리할 개념
✅ remote의 host(`github.com` vs `git.{your_company_name}.com`)가 “어떤 키로 인증할지”를 결정
- Host를 도메인 그대로 쓰면 URL이 표준형이라 깔끔하지만,
- 같은 도메인에서 “서로 다른 계정”을 나누려면 alias/sshCommand 같은 추가 장치가 필요

---

### 6단계) 최종 확인(전환/격리 검증)

#### 6-1) 커밋 작성자 전환 확인
```bash
git -C ~/Document/dev/otherProject/some-repo config user.email
git -C ~/somewhere/some-repo config user.email
```

#### 6-2) 인증(키) 확인
```bash
ssh -T git@github.com
ssh -T git@git.{your_company_name}.com
```

#### 6-3) 디버그(필요 시)
```bash
ssh -vT git@github.com
ssh -vT git@git.{your_company_name}.com
```

#### 이 단계에서 정리할 개념
✅ `ssh -T`는 “호스트별 키 매핑”이 제대로 되는지 가장 빠른 확인법  
✅ `Permission denied (publickey)`면 2단계 해결 순서를 그대로 따라가면 됨

---

## 4. 최종 요약 체크리스트
- [ ] SSH 키 2개 생성(ed25519) + 각 서버 계정에 public key 등록 완료
- [ ] (선택) `ssh-add`로 키 로드했고 `ssh-add -l`로 확인 가능
- [ ] `~/.ssh/config`에 `Host github.com`(기본키), `Host git.{your_company_name}.com`(a키) 설정 + `IdentitiesOnly yes`
- [ ] `ssh -T git@github.com` / `ssh -T git@git.{your_company_name}.com` 성공
- [ ] `~/.gitconfig`에는 **기본 계정 user.name/email만** 존재
- [ ] `~/.gitconfig`에는 `includeIf "gitdir:~/Document/dev/otherProject/"` **규칙만** 존재 (a 정보 없음)
- [ ] `~/.gitconfig-otherProject-a`에는 a 계정 `user.name/email`만 존재
- [ ] otherProject의 repo remote host가 실제로 `git.{your_company_name}.com`인지 확인
- [ ] GitHub로 클론(`git@github.com:...`)하면 기본 계정 키로 인증되는 것까지 이해 완료
