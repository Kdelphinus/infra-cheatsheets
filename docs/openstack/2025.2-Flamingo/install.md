# 🚀 OpenStack Flamingo (2025.2) 통합 설치 가이드

## 📋 0. OS별 사전 준비 (OS Preparation)

OS에 따라 초기 설정이 다릅니다. 특히 **Rocky Linux는 SELinux 설정이 필수**입니다.

### 🟠 [Rocky Linux] 필수 설정 (SELinux 해제)

RedHat 계열은 SELinux가 켜져 있으면 Kolla 배포 시 권한 문제로 실패합니다.

> 보안 정책 상 SELinux 비활성화가 불가능한 환경에서는, Kolla-Ansible SELinux 대응 가이드를 별도로 참고해야 합니다.

```bash
# 1. SELinux를 Permissive 모드로 변경 (일시적)
sudo setenforce 0

# 2. 영구 설정 (재부팅 후에도 유지되도록)
sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config

# 3. EPEL 저장소 추가 (필수 패키지 설치용)
sudo dnf install -y epel-release
sudo dnf update -y
```

### 🟣 [Ubuntu] 필수 설정

Ubuntu는 패키지 업데이트만 하면 됩니다.

```bash
sudo apt update && sudo apt upgrade -y
```

-----

## 🛠️ 1. 의존성 패키지 설치 (Install Dependencies)

Python 가상환경을 만들기 위한 기초 도구들을 설치합니다.

### 🟠 [Rocky Linux]

```bash
# 개발 도구 및 파이썬 라이브러리 설치
sudo dnf install -y git python3-devel libffi-devel gcc openssl-devel python3-libselinux python3-pip python3-libselinux-devel
```

### 🟣 [Ubuntu]

```bash
# 개발 도구 및 파이썬 라이브러리 설치
sudo apt install -y git python3-dev libffi-dev gcc libssl-dev python3-venv libdbus-glib-1-dev python3-dbus
```

-----

## 🐍 2. 가상환경 구성 (Virtual Environment) - [공통]

여기서부터는 OS 상관없이 동일합니다. **시스템 파이썬을 더럽히지 않기 위해 가상환경 사용이 필수**입니다.

```bash
# 1. 가상환경 생성 (홈 디렉터리에 venv 생성)
python3 -m venv ~/venv

# 2. 가상환경 활성화 ★(작업할 때마다 매번 실행 필수)★
source ~/venv/bin/activate

# (프롬프트 앞에 (venv)가 떴는지 확인하세요)

# 3. pip 최신화 (오류 방지)
pip install -U pip
```

-----

## 📦 3. Kolla-Ansible 설치 (Install) - [공통]

```bash
# 1. Kolla-Ansible Flamingo 버전 설치 (master 브랜치)
pip install git+https://opendev.org/openstack/kolla-ansible@master

# 버전 지정
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2025.2

# 2. 설정 디렉터리 생성 및 권한 부여
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla

# 3. 설정 파일 복사 (globals.yml, passwords.yml)
cp -r ~/venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/

# 4. 인벤토리 파일 복사 (멀티노드용)
cp ~/venv/share/kolla-ansible/ansible/inventory/multinode .
```

-----

## ⚙️ 4. 설정 파일 수정 (`globals.yml`) - [OS별 차이점]

`vi /etc/kolla/globals.yml`을 열어서 수정합니다. OS에 따라 **`kolla_base_distro`** 값을 다르게 줘야 합니다.

```yaml
---
# [중요] OS에 따라 선택하세요
# Ubuntu 사용 시:
kolla_base_distro: "ubuntu"
# Rocky Linux 사용 시:
# kolla_base_distro: "rocky"

# [공통] 오픈스택 버전 (Flamingo 대응)
# openstack_release: "master"  <-- 주석 그대로 두거나, Docker 태그 명시

# [공통] 네트워크 설정 (사용자 환경에 맞춰서 수정할 것)
# eno2에는 IP를 설정하지 말고, 스위치에 외부망 VLAN/Untaged 연결 필수
network_interface: "eno1"           # 관리망 (IP 10.10.10.60)
neutron_external_interface: "eno2"  # 외부망 (IP 없음)

# [공통] VIP 주소 (관리망 대역 내 미사용 IP, 같은 IP 사용 시 HAProxy 충돌 발생 가능성 높음)
kolla_internal_vip_address: "10.10.10.60"  # 기존 설정은 controll ip와 동일하게 설정되어 있음

# [공통] 주요 서비스 활성화
enable_cinder: "yes"
enable_heat: "yes"
enable_horizon: "yes"
# enable_prometheus: "yes" # 모니터링 필요 시
```

-----

## 📝 5. 인벤토리 및 비밀번호 설정 - [공통]

### 5.1 비밀번호 생성

```bash
kolla-genpwd
# 생성 후 Horizon Admin 비밀번호 확인해두기
grep keystone_admin_password /etc/kolla/passwords.yml
```

### 5.2 SSH 접속 허용

Master 노드에서 Compute 노드 등으로 비밀번호 없이 SSH 접속이 가능하도록 설정합니다.

```bash
# 1. SSH 공개 키를 배포 대상 서버에 복사 (각 노드 IP별로 반복 수행)
ssh-copy-id -i ~/.ssh/id_rsa.pub <USER_ID>@<TARGET_NODE_IP>

# 2. 접속 테스트 (암호 없이 로그인 되면 성공)
ssh <USER_ID>@<TARGET_NODE_IP>
```

### 5.3 인벤토리 수정 (`vi multinode`)

`multinode` 파일을 열어 배포 대상 서버 정보를 입력합니다.

```bash
# 3. 인벤토리 수정
sudo vi multinode
```

**파일 내용 수정 예시:**

```ini
[control]
<MASTER_IP>

[network]
<MASTER_IP>

[compute]
# Compute 노드 IP와 계정 정보를 기입
<TARGET_NODE_IP> ansible_user=<USER_ID> ansible_become=true
```

타켓 IP에 입력한 ID로 접속하여 sudo 권한을 얻겠다는 의미입니다.

### 5.4 통신 확인

```bash
# 4. Ansible Ping 테스트
ansible -i multinode all -m ping
```

### 5.5 Ansible 의존성 설치 (필수)

```bash
kolla-ansible install-deps
```

### 5.6 그 외

```bash
# 타임싱크 확인
timedatectl

# 호스트명 확인
hostname

# /etc/hosts에 모든 노드 등록 권장
vi /etc/hosts
```

-----

## 🚀 6. 배포 실행 (Deploy) - [공통]

이제 실제 설치를 진행합니다.

```bash
# 1. Bootstrap (기초 공사: Docker 설치 등)
kolla-ansible -i ./multinode bootstrap-servers

# 2. Prechecks (사전 검사: 설정 오류 확인)
kolla-ansible -i ./multinode prechecks
# -> 여기서 "SUCCESS"가 떠야만 다음으로 넘어갑니다.

# 3. Deploy (본 게임: 컨테이너 배포)
kolla-ansible -i ./multinode deploy
```

-----

## ✅ 7. 클라이언트 설정 (Post-Deploy) - [공통]

배포가 끝나면 OpenStack 명령어를 쓰기 위한 도구를 세팅합니다.

```bash
# 1. OpenStack 클라이언트 설치
pip install python-openstackclient

# 2. 관리자 인증 파일 생성 (admin-openrc.sh)
kolla-ansible post-deploy

# 3. 인증 로드 및 테스트
source /etc/kolla/admin-openrc.sh
openstack service list

# 4. Nova와 Compute 등록 확인
openstack hypervisor list
openstack compute service list
```

-----

## 💡 DevOps 엔지니어의 조언: Ubuntu vs Rocky 선택 가이드

- **Ubuntu 24.04:**
  - **장점:** OpenStack 커뮤니티의 \*\*표준(De-facto)\*\*입니다. 트러블슈팅 자료가 제일 많고, `apt`가 빠릅니다.
  - **추천:** 특별한 이유가 없다면 Ubuntu로 가세요.
- **Rocky Linux 9/10:**
  - **장점:** RHEL(RedHat) 기반이라 엔터프라이즈 환경에서 선호됩니다. 안정성이 높습니다.
  - **단점:** SELinux 때문에 초기 설정이 귀찮고, 패키지 이름이 Ubuntu랑 달라서 가끔 헷갈립니다.
  - **추천:** 사내 정책상 RHEL 계열을 써야만 할 때 선택하세요.
