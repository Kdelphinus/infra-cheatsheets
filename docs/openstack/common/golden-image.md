# OpenStack 골든 이미지(Golden Image) 생성 가이드

OpenStack 환경(Nova, Glance)에서 인스턴스 배포 시 즉시 사용 가능한 깨끗한 Cloud Image를 만들기 위한 절차입니다.

> **필수 전제 조건**:
>
> - 작업은 로컬 KVM/QEMU 환경이나 OpenStack 내 임시 인스턴스에서 수행합니다.
> - 모든 명령은 **root 권한**으로 실행합니다.

## 1. 베이스 이미지 다운로드

클라우드용으로 미리 빌드된 Cloud Image(qcow2, img)를 받습니다.

```bash
mkdir -p ~/golden-image-work
cd ~/golden-image-work
```

**1. Ubuntu 24.04 LTS (Noble):**

```bash
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img -O ubuntu-24.04-base.qcow2
```

**2. Rocky Linux 9:**

```bash
wget https://dl.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud-Base.latest.x86_64.qcow2 -O rocky-9-base.qcow2
```

**3. AlmaLinux 9:**

```bash
wget https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2 -O AlmaLinux-9-base.qcow2
```

**4. CentOS Stream 9:**

```bash
wget https://cloud.centos.org/centos/9-stream/x86_64/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2 -O CentOS-Stream-9-base.qcow2
```

**5. Debian 12:**

```bash
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2 -O debian-12-base.qcow2
```

```bash
# `unable to resolve host addree` 오류 발생 시, dns 서버 추가
echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf
```

## 2. 이미지 접근 권한 설정

Cloud Image는 기본적으로 root 비밀번호와 SSH 접속이 막혀있습니다.

따라서 로컬 KVM에서 부팅 및 작업을 위해 `libguestfs-tools` 와 `virt-customize` 도구를 설치하여 비밀번호를 강제로 설정합니다.

```bash
# 도구 설치 (Ubuntu 기준)
sudo apt update
sudo apt install -y libguestfs-tools qemu-kvm virt-manager

# 이미지에 root 비밀번호 설정 (예: root1234)
# 주의: 이 비밀번호는 나중에 스크립트 실행 후 제거할 것입니다.
sudo virt-customize -a ubuntu-24.04-base.qcow2 --root-password password:root1234
```

## 3. KVM으로 VM 부팅 및 접속

### 3.1 Ubuntu/Debian

```bash
# Ubuntu는 기본 용량이 적어 용량을 늘려줍니다.
qemu-img resize ubuntu-24.04-base.qcow2 +10G

# KVM으로 부팅 (메모리 4G, CPU 2 Core)
sudo kvm -m 2048 -smp 2 -hda ubuntu-24.04-base.qcow2 -net nic -net user,hostfwd=tcp::2222-:22 -nographic
```

위에서 `login:` 화면이 뜨면 위에서 설정한 비밀번호로 접속합니다.

편하게 붙여넣기 위해서 ssh 접속 설정을 활성화해줍니다. (기존 터미널은 글자가 밀립니다.)

기본 이미지에 키가 없는 경우가 있어서 키부터 생성합니다.

```bash
# 출력 결과가 없다면 키를 만들어줘야 합니다.
ls -l /etc/ssh/ssh_host_*
```

```bash
# 키 생성
ssh-keygen -A

# sshd 설정 중 아래 두 항목을 yes로 변경
vi /etc/ssh/sshd_config
...
PermitRootLogin yes
PasswordAuthentication yes
...

# Override 하는 설정 파일 제거
rm -f /etc/ssh/sshd_config.d/60-cloudimg-settings.conf

# ssh 재시작
systemctl restart ssh

# 설정 확인
sshd -T | grep passwordauthentication
```

```bash
# 상태 확인
systemctl status ssh

# 혹시 disable이라면 enable로 변경
systemctl enable ssh
```

아래 명령 실행 시, ens3과 같은 네트워크가 DOWN 상태라면 아래 조치를 취합니다.

```bash
ip a
```

ens3은 실제 네트워크 이름으로 변경해야 합니다.

```bash
# 인터페이스 켜기
ip link set ens3 up

# IP 주소 할당(10.0.2.15는 KVM 기본 IP)
ip addr add 10.0.2.15/24 dev ens3

# 게이트웨이 연결
ip route add default via 10.0.2.2

# DNS 설정
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

```bash
# 네트워크 연결 확인
ping -c 3 8.8.8.8
```

### 3.2 Rocky/RHEL

Rocky는 CPU의 여러 기능을 요구합니다. 따라서 아래와 같이 호스트 CPU를 그대로 VM에 전달하도록 `-cpu host` 옵션을 사용합니다.

```bash
sudo kvm -m 4096 -smp 2 -cpu host -hda rocky-9-base.qcow2 -net nic -net user,hostfwd=tcp::2222-:22 -nographic
```

ssh 접속을 위해 설정을 변경합니다.

```bash
# sshd 설정 중 아래 두 항목을 yes로 변경
vi /etc/ssh/sshd_config

...
PermitRootLogin yes
PasswordAuthentication yes
...

# ssh 재시작
systemctl restart sshd
```

새로운 터미널을 열어 ssh로 접속합니다.

```bash
ssh root@localhost -p 2222
```

> 작업 중 VM을 멈추거나 종료하고 싶다면 `Ctrl + A` 키를  눌렀다가 떼고, 바로 `x` 키를 누르면 KVM이 강제 종료됩니다.

### KVM 관련 오류

#### 1 ) `Lock` 오류

KVM 부팅 중 오류가 발생하거나, `lock` 오류가 발생한다면 프로세스를 죽이고 다시 위 과정을 진행해야 합니다.

```bash
# 실행 중인 kvm 프로세스 확인
ps -ef | grep kvm
```

```bash
# 'ubuntu-24.04-base.qcow2'가 포함된 모든 프로세스 강제 종료
pkill -f ubuntu-24.04-base.qcow2

# 만약 위에 명령이 실행되지 않는다면
pkill -9 <PID>
```

#### 2) KVM 용량을 늘려도 VM 용량이 동일한 경우

현재 아래 명령을 통해 `sda` 는 늘린 용량인데, `sda1` 이 여전히 그대로인 것을 확인할 수 있습니다.

```bash
# 현재 상황 확인
lsblk
```

우선 파티션을 디스크 끝까지 늘립니다.

```bash
growpart /dev/sda 1
```

늘어난 파티션에 맞춰서 파일시스템을 확장합니다.

```bash
resize2fs /dev/sda1
```

용량이 늘어났는지 확인합니다.

```bash
df -h
```

## 4. VM 내부 설정 변경

OpenStack의 Horizon 콘솔 로그 확인 및 하이퍼바이저 통신을 위해 아래 설정이 모든 이미지에 적용되어야 합니다.

### 4.1 Serial Console 활성화 (OpenStack 필수)

OpenStack 대시보드에서 로그를 보려면 커널 파라미터 수정이 필요합니다.

**Ubuntu/Debian:**

```bash
# /etc/default/grub 파일 수정
sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT.*/GRUB_CMDLINE_LINUX_DEFAULT="console=tty0 console=ttyS0,115200n8 earlyprintk=ttyS0,115200 rootdelay=300"/' /etc/default/grub

# 수정 내용 적용
update-grub
```

**RHEL/Rocky/CentOS:**

```bash
# /etc/default/grub 파일 수정
cat > /etc/default/grub << 'EOF'
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto console=tty1 console=ttyS0,115200n8"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true
EOF

# 수정 내용 적용
grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 4.2 기본 패키지 설치

VMware Tools 대신 **QEMU Guest Agent**를 사용합니다.

**Ubuntu/Debian:**

```bash
# DNS 서버 수동 등록
echo "nameserver 8.8.8.8" > /etc/resolv.conf

# 필수 패키지 설치
apt update
apt install -y qemu-guest-agent cloud-init
```

**Rocky/RHEL:**

```bash
# DNS 서버 수동 등록
echo "nameserver 8.8.8.8" > /etc/resolv.conf

# 필수 패키지 설치
yum install -y qemu-guest-agent cloud-init
```

-----

## 5. 옵션 소프트웨어 설치

### 5.1 옵션: Docker 설치

Docker를 포함하는 경우에만 이 단계를 수행하세요.

**Ubuntu/Debian:**

```bash
# 1. 필수 패키지 및 GPG Key 설정
apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# 2. 저장소 추가
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update

# 3. Docker 설치
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 4. 활성화
systemctl enable --now docker

# -------------------------------------------------------
# [GPU 이미지인 경우만] NVIDIA Container Toolkit 설치
# -------------------------------------------------------

# 5. 저장소 추가 및 설치
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

apt-get update
apt-get install -y nvidia-container-toolkit

# 6. Docker 설정 및 재시작
nvidia-ctk runtime configure --runtime=docker
systemctl restart docker
```

**Rocky/RHEL:**

```bash
# 1. Docker 저장소 추가
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 2. Docker 엔진 설치
dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 옵션. Rocky 10의 경우 아래 패키지도 설치
dnf install -y iptables-nft iptables-services

# 3. Docker 실행 및 자동 시작 설정
systemctl enable --now docker

# -------------------------------------------------------
# [GPU 이미지인 경우만] NVIDIA Container Toolkit 설치
# (Docker가 GPU를 쓰게 해주는 필수 도구입니다)
# -------------------------------------------------------

# 4. NVIDIA Toolkit 저장소 추가
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
  tee /etc/yum.repos.d/nvidia-container-toolkit.repo

# 5. Toolkit 설치
dnf install -y nvidia-container-toolkit

# 6. Docker가 NVIDIA Runtime을 쓰도록 설정 (중요!)
# 이 명령어가 /etc/docker/daemon.json 파일을 자동으로 수정해줍니다.
nvidia-ctk runtime configure --runtime=docker

# 7. 적용을 위해 Docker 재시작
systemctl restart docker
```

### 5.2 옵션: NVIDIA GPU 드라이버 설치

**GPU 버전 이미지**를 생성할 때만 이 단계를 수행하세요. CPU 전용 이미지는 이 단계를 건너뜁니다.

> **전제**: 인스턴스에 이미 NVIDIA GPU가 Passthrough 되어 있거나, 로컬에서 드라이버 빌드 환경이 갖춰져야 합니다.

**Ubuntu/Debian:**

```bash
# 1. Nouveau 블랙리스트 처리
cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF

# 옵션. RTX 계열의 개인 GPU 사용 시 MSI 비활성화
echo "options nvidia NVreg_EnableMSI=0" | sudo tee /etc/modprobe.d/nvidia.conf

# 2. 변경 사항 커널에 적용
update-initramfs -u

# 3. 빌드 의존성 패키지 설치
apt-get install -y build-essential linux-headers-$(uname -r) pkg-config libglvnd-dev

# 4. NVIDIA 드라이버 다운로드
cd /root
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/550.127.05/NVIDIA-Linux-x86_64-550.127.05.run
chmod +x NVIDIA-Linux-x86_64-*.run

# 5. 드라이버 설치 (하드웨어 없이 설치)
# 나중에 로그인해서 'install-gpu'만 치면 설치되도록 설정
echo "alias install-gpu='/root/NVIDIA-Linux-x86_64-*.run --silent --dkms && echo \"설치 완료! nvidia-smi를 입력해보세요.\"' " >> /root/.bashrc
```

**Rocky/RHEL:**

```bash
# 1. Nouveau(오픈소스 드라이버) 블랙리스트 처리
cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF

# 옵션. RTX 계열의 개인 GPU 사용 시 MSI 비활성화
echo "options nvidia NVreg_EnableMSI=0" | sudo tee /etc/modprobe.d/nvidia.conf

# 2. 변경 사항 커널에 적용 (initramfs 갱신)
dracut --force

# 3. 빌드 의존성 패키지 설치
# (드라이버 컴파일을 위해 gcc, make, kernel source 등이 필요함)
yum groupinstall -y "Development Tools"
yum install -y epel-release
yum update -y  # 커널 최신화 (중요)
yum install -y kernel-devel kernel-headers dkms gcc make wget pciutils

# 드라이버 다운로드
cd /root
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/550.127.05/NVIDIA-Linux-x86_64-550.127.05.run
chmod +x NVIDIA-Linux-x86_64-*.run

# 5. 드라이버 설치 (하드웨어 없이 설치)
# --silent: 무인 설치
# 나중에 로그인해서 'install-gpu'만 치면 설치되도록 설정
echo "alias install-gpu='/root/NVIDIA-Linux-x86_64-*.run --silent --dkms && echo \"설치 완료! nvidia-smi를 입력해보세요.\"' " >> /root/.bashrc
```

-----

## 6. VM 정리 작업

`Cloud-init` 이 정상적으로 작동하도록 설정들을 초기화 합니다.

**Ubuntu/Debian:**

```bash
# 1. DataSource 설정
cat > /etc/cloud/cloud.cfg.d/90_datasource.cfg << 'EOF'
datasource_list: [ OpenStack, ConfigDrive, None ]
EOF

# 2. SSH 보안 설정
# (Ubuntu는 서비스명이 'ssh' 입니다)
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart ssh

# 3. 네트워크 설정 초기화
# Netplan 설정 삭제
rm -f /etc/netplan/*.yaml
# 구버전 인터페이스 설정 삭제
rm -f /etc/network/interfaces.d/*

# 4. 고유 식별자 및 Cloud-init 초기화
rm -f /etc/udev/rules.d/70-persistent-net.rules
rm -f /etc/ssh/ssh_host_*key*
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id

# Cloud-init 데이터 삭제
rm -rf /var/lib/cloud/*
rm -rf /var/log/cloud-init*

# 5. 패키지 캐시 정리
apt-get clean
rm -rf /var/lib/apt/lists/*
```

**Rocky/RHEL:**

```bash
# 1. DataSource 설정 (OpenStack 최우선)
cat > /etc/cloud/cloud.cfg.d/90_datasource.cfg << 'EOF'
datasource_list: [ OpenStack, ConfigDrive, None ]
EOF

# 2. SSH 보안 설정 (클라우드 표준: 패스워드 인증 비활성화)
# (Root 로그인은 보안 정책에 따라 'prohibit-password' 또는 'yes'로 설정)
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd

# 2.1 cloud-init 단계에서도 root 로그인 차단
echo "disable_root: true" >> /etc/cloud/cloud.cfg

# 3. 네트워크 설정 초기화 (기존 IP 정보 삭제)
# NetworkManager 연결 정보 삭제
rm -f /etc/NetworkManager/system-connections/*.nmconnection
# 구버전 스크립트가 있다면 삭제 (ifcfg-lo 제외)
rm -f /etc/sysconfig/network-scripts/ifcfg-e*

# 4. 고유 식별자 및 Cloud-init 초기화
rm -f /etc/udev/rules.d/70-persistent-net.rules
rm -f /etc/ssh/ssh_host_*key*
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id

# Cloud-init 데이터 삭제 (재부팅 시 새로 받기 위함)
rm -rf /var/lib/cloud/*
rm -rf /var/log/cloud-init*

# 5. 패키지 캐시 정리
yum clean all
rm -rf /var/cache/yum
```

-----

## 7. 최종 정리 및 이미지 변환

모든 설정이 끝났으면 마지막으로 cloud-init을 초기화하고 VM을 종료합니다.

### 7.1 최종 명령어

```bash
# 로그 파일 비우기
find /var/log -type f -exec truncate -s 0 {} \;

# cloud-init 정리 (가장 중요)
cloud-init clean --logs

# history 삭제
history -c

# 시스템 종료 (재부팅 하지 마세요!)
shutdown -h now
```

### 7.2 Glance 이미지 업로드 (OpenStack CLI)

VM 이미지 파일(qcow2 등)을 OpenStack Glance에 업로드합니다.

```bash
# 1. 이미지 압축 (Shrink)
# 원본(base) -> 최종본(golden)
qemu-img convert -c -O qcow2 ubuntu-24.04-base.qcow2 ubuntu-24.04-golden.qcow2

# 2. 용량 확인 (확 줄어들었는지 확인)
ls -lh ubuntu-24.04-golden.qcow2
```

**CPU 전용 이미지:**

```bash
openstack image create "Ubuntu-24.04-Golden" \
  --file ubuntu-24.04-golden.qcow2 \
  --disk-format qcow2 \
  --container-format bare \
  --public \
  --property hw_machine_type=q35 \
  --property hw_qemu_guest_agent=yes
```

**GPU 포함 이미지 (메타데이터 중요):**
OpenStack Nova가 GPU를 적절히 스케줄링하도록 메타데이터를 추가할 수 있습니다.

```bash
openstack image create "Ubuntu-24.04-Golden-GPU" \
  --file ubuntu-24.04-golden.qcow2 \
  --disk-format qcow2 \
  --container-format bare \
  --public \
  --property hw_video_model=vga \
  --property hw_machine_type=q35 \
  --property hw_qemu_guest_agent=yes
```

> 만약 RTX 계열의 개인용 GPU 사용한다면 이미지 혹은 flavor에 `--property hw:kvm_hidden=true` 값을 넣어주어야 합니다.

## 8. 옵션: GPU 드라이버 설치

GPU 인스턴스 생성하고, 플로팅 IP를 연결하여 VM 내부에 접속한 뒤, 아래 명령어를 실행시킵니다.

```bash
# root 계정 사용
sudo su -
```

```bash
# 사전에 받아둔 nvidia 드라이버 설치
install-gpu
```

### RTX 계열 GPU 사용 시 문제점(Reset Bug)

RTX 계열과 같은 소비자용 GPU는 가상화 환경을 고려하지 않기에 FLR(Function Level Reset) 기능을 지원하지 않습니다.
따라서 GPU 인스턴스를 종료하거나 재부팅하면, GPU 하드웨어가 초기화되지 않고 Stuck 상태로 남아 데이터 찌꺼기를 가지고 있다가 VM의 다음 동작을 방해하고, 이를 호스트가 응답이 없는 것으로 인식하고 인스턴스를 PAUSED 상태로 강제 정지시켜버립니다.

위에서 RTX 계열 GPU를 위한 설정을 추가했지만, 이것만으로 문제를 막을 수는 없습니다.
FLR 기능이 없는 한 재부팅 혹은 삭제 시 높은 확률로 하드웨어 리셋 실패가 발생하기에 RTX 계열 GPU를 사용하지 않는 것을 권장합니다.

테스트 용도로 사용 예정 시, [FLR 동작 임시 적용법](https://kdelphinus.github.io/infra-cheatsheets/openstack/common/init-gpu-node/#rtx-gpu)을 참고바랍니다.
