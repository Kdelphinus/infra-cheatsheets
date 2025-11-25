# WSL 속도 향상 설정

## 🚀 해결 방법 1: DNS 서버 수동 고정 (가장 효과적 ⭐)

WSL2가 윈도우 호스트를 거쳐서 DNS를 조회하는 과정이 느리기 때문에, 구글(8.8.8.8)이나 클라우드플레어(1.1.1.1) DNS를 직접 바라보게 설정하면 속도가 비약적으로 빨라집니다.

### 1. 자동 생성 설정 끄기 (`/etc/wsl.conf`)

WSL 내부 터미널에서 아래 명령어를 입력해 설정 파일을 만듭니다.

```bash
# wsl.conf 파일 생성/수정
sudo nano /etc/wsl.conf
```

아래 내용을 복사해서 붙여넣으세요. (이미 내용이 있다면 `[network]` 부분에 추가)

```ini
[network]
generateResolvConf = false
```

(`Ctrl + O` 저장, `Enter`, `Ctrl + X` 종료)

### 2. DNS 파일 직접 수정 (`/etc/resolv.conf`)

기존의 심볼릭 링크를 끊고 직접 DNS를 입력합니다.

```bash
# 기존 파일 삭제 (심볼릭 링크임)
sudo rm /etc/resolv.conf

# 새 파일 생성 및 DNS 입력
sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
sudo bash -c 'echo "nameserver 1.1.1.1" >> /etc/resolv.conf'
```

### 3. WSL 재시작 (PowerShell에서)

```powershell
wsl --shutdown
```

다시 WSL을 켜서 `apt update` 등을 해보면 훨씬 빨라진 것을 느낄 수 있습니다.

-----

## 🚀 해결 방법 2: "Mirrored" 모드 사용 (Windows 11 필수 ⭐⭐)

최신 Windows 11(22H2 이상)과 최신 WSL 버전을 쓰고 계시다면, \*\*"미러링 모드"\*\*를 켜는 것이 **게임 체인저**입니다. NAT를 거치지 않고 호스트(Windows)의 네트워크 인터페이스를 그대로 공유하므로 속도가 네이티브 급으로 빨라집니다.

### 1. 윈도우 사용자 폴더에 설정 파일 생성

윈도우 탐색기 주소창에 `%UserProfile%`을 입력하고 엔터를 칩니다.
여기에 **`.wslconfig`** 라는 파일을 만들고(메모장 이용) 아래 내용을 넣으세요.

```ini
[wsl2]
networkingMode=mirrored
dnsTunneling=true
firewall=true
autoProxy=true
```

- **`networkingMode=mirrored`**: NAT를 없애고 윈도우 IP를 그대로 씁니다. (속도 대폭 향상)
- **`dnsTunneling=true`**: DNS 요청을 가상화 계층 대신 터널링으로 처리해 딜레이를 줄입니다.

### 2. 적용

PowerShell에서 `wsl --shutdown` 후 다시 시작.

-----

## 🚀 해결 방법 3: IPv6 비활성화 (특정 통신사/환경용)

`apt-get`이나 `curl`이 처음에 멈칫하다가 진행되는 경우, IPv6 연결을 시도하다가 타임아웃이 나서 IPv4로 넘어가는 딜레이일 수 있습니다.

**WSL 내부에서 실행:**

```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
```

(이게 효과가 있다면 `/etc/sysctl.conf`에 영구 등록하세요.)

-----

## 추가 정보

1. **Windows 11** 사용자라면 \*\*[방법 2 (Mirrored Mode)]\*\*가 가장 깔끔하고 성능이 좋습니다. (`apt` 속도뿐만 아니라 로컬호스트 접속 문제도 해결됨)
2. **Windows 10** 사용자거나 Mirrored 모드가 불안정하다면 \*\*[방법 1 (DNS 고정)]\*\*만 적용해도 충분히 빨라집니다.
