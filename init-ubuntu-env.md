# Ubuntu 기본 세팅

- Ubuntu 24.04 기준

## Index

- [Oh My Zsh 설치](#oh-my-zsh-설치)
- [npm 설치](#npm-설치)

## Oh My Zsh 설치

### 1. Oh My Zsh 설치

```bash
# zsh 설치
sudo apt install zsh
```

```bash
# oh-my-zsh 설치
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### 2. Powerlevel10k 및 플러그인 설치

```bash
# Powerlevel10k
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k

# zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-autosuggestions.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

# zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

### 3. ~/.zshrc 파일 수정

```bash
# ZSH_THEME="robbyrussell"
ZSH_THEME="powerlevel10k/powerlevel10k"

plugins=(
    git
    zsh-autosuggestions
    zsh-syntax-highlighting
)
```

### 4. 셸 새로고침

```bash
source ~/.zshrc
```

> 설정: source ~/.zshrc를 실행하면, **설정 마법사(p10k configure)**가 자동으로 실행됩니다. 질문에 답하면서 원하는 아이콘과 모양을 선택하면 됩니다.
> 필수: 이 테마를 제대로 사용하려면 Nerd Fonts (예: MesloLGS NF)를 Tabby 터미널의 글꼴로 설정해야 합니다.

## npm 설치

### 1. NVM 설치

```bash
# NVM 설치
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# NVM 활성화
source ~/.zshrc
```

### 2. Node.js 설치

```bash
# Node.js LTS 버전 설치
nvm install --lts
```

```bash
# 설치 확인
node -v
npm -v
```
