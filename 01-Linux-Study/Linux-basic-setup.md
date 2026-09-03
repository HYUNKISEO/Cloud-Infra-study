# ⚡ [ALL-IN-ONE] 리눅스 인프라 초기 설정 및 환경 자동화 매뉴얼

새 깡통 리눅스 서버(AWS EC2, Ubuntu, CentOS, 가상머신 등) 접속 직후 단 한 줄의 명령어(One-Liner)로 **타임존 동기화 + 2GB 스왑 메모리 생성 + 필수 패키지 자동 설치 + 나만의 단축어(`alias`) 적용**을 한 번에 구축하는 통합 매뉴얼입니다.

---

## 1. 깃허브 사전 준비 (저장소: `my-dotfiles`)

깃허브에 **`my-dotfiles`** 이름으로 저장소(Repository)를 생성하고, 루트 디렉터리에 아래 **3개 파일**을 위치시킵니다.

### ① `.bash_aliases` (내 전용 단축어 모음집)
```bash
# 자주 쓰는 기본 리눅스 단축어
alias d='echo 떡볶이'
alias ll='ls -al'
alias c='clear'

# 실무 인프라 트러블슈팅 단축어
alias ports='ss -tulpn'
alias ghost='lsof | grep deleted'

# 자주 쓰는 Git 명령어 단축어
alias gs='git status'
alias gp='git pull'
```

### ② `packages.txt` (선언적 패키지 관리 목록)
```text
# 네트워크 & 모니터링
net-tools
htop
curl
lsof

# 에디터 및 필수 유틸리티
vim
jq
tmux
git
```

### ③ `setup.sh` (올인원 자동화 스크립트)
```bash
#!/bin/bash
GITHUB_ID="내아이디"

echo "🚀 [1/5] 타임존 설정 (Asia/Seoul) 및 NTP 시간 자동 동기화..."
sudo timedatectl set-timezone Asia/Seoul
sudo timedatectl set-ntp true

echo "🚀 [2/5] 스왑 메모리(Swap 2GB) 자동 생성 (OOM 다운 현상 방지)..."
if [ ! -f /swapfile ]; then
    sudo fallocate -l 2G /swapfile || sudo dd if=/dev/zero of=/swapfile bs=1M count=2048
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
fi

echo "🚀 [3/5] my-dotfiles 저장소 동기화..."
if [ ! -d "$HOME/my-dotfiles" ]; then
    git clone [https://github.com/$GITHUB_ID/my-dotfiles.git](https://github.com/$GITHUB_ID/my-dotfiles.git) ~/my-dotfiles
else
    (cd ~/my-dotfiles && git pull --quiet)
fi

echo "🚀 [4/5] packages.txt 기반 필수 패키지 자동 설치..."
PACKAGES_FILE="$HOME/my-dotfiles/packages.txt"
if [ -f "$PACKAGES_FILE" ]; then
    PKG_LIST=$(grep -vE '^\s*#|^\s*$' "$PACKAGES_FILE" | tr '\n' ' ')
    if command -v apt-get &> /dev/null; then
        sudo apt-get update -qq && sudo apt-get install -y -qq $PKG_LIST > /dev/null
    elif command -v yum &> /dev/null; then
        sudo yum install -y -q $PKG_LIST > /dev/null
    fi
fi

echo "🚀 [5/5] .bashrc 연결 및 터미널 동기화 등록..."
if ! grep -q "source ~/my-dotfiles/.bash_aliases" ~/.bashrc; then
    echo "" >> ~/.bashrc
    echo "# Custom Dotfiles & Aliases" >> ~/.bashrc
    echo "source ~/my-dotfiles/.bash_aliases" >> ~/.bashrc
    echo "(cd ~/my-dotfiles && git pull --quiet)" >> ~/.bashrc
fi

echo "✅ [완료] 타임존(KST), 스왑(2GB), 필수 패키지, 단축어 자동화 세팅이 완벽히 적용되었습니다!"
```

---

## 2. 새 리눅스 서버에서 실행 방법 (One-Liner)

새 깡통 서버에 접속하자마자 아래 **한 줄 명령어**만 실행하면 모든 초기 환경 구축이 완료됩니다.  
*(※ `내아이디` 부분을 본인의 깃허브 ID로 변경 후 실행)*

```bash
curl -sL [https://raw.githubusercontent.com/내아이디/my-dotfiles/main/setup.sh](https://raw.githubusercontent.com/내아이디/my-dotfiles/main/setup.sh) | bash && source ~/.bashrc
```

---

## 3. 핵심 동작 원리 & 실무 인사이트

1. **시간 동기화 (KST)**: 로그 시각 불일치 및 SSL/NTP 검증 실패 예방.
2. **2GB 스왑 메모리 방어막**: 소형 인스턴스의 메모리 부족으로 인한 `OOM Killer` 서버 다운 현상 방지.
3. **선언적 패키지 관리 (`packages.txt`)**: 스크립트 수정 없이 텍스트 파일만 수정(`git push`)해 환경 업데이트 (IaC/Docker 핵심 원리).
4. **무한 자동 동기화**: 터미널 접속 시마다 최신 단축어를 `git pull` 받아 모든 서버 환경 동기화.
