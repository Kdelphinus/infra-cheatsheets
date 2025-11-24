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

```bash
# KVM으로 부팅 (메모리 2G, CPU 2 Core)
sudo kvm -m 2048 -smp 2 -hda ubuntu-24.04-base.qcow2 -net nic -net user,hostfwd=tcp::2222-:22 -nographic
```

위에서 `login:` 화면이 뜨면 위에서 설정한 비밀번호로 접속합니다.

만약 아래 `echo` 명령이 깨져서 나온다면, ssh 관련 설정을 변경하여 ssh로 접속해야 합니다.

```bash
echo "This is a test to see if copy and paste works correctly in the KVM console."
```

```bash
# sshd 설정 중 아래 두 항목을 yes로 변경
vi /etc/ssh/sshd_config

...
PermitRootLogin yes
PasswordAuthentication yes
...
```

```bash
# Ubuntu/Debian
systemctl restart ssh

# Rocky/RHEL
systemctl restart sshd
```

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

#### 2) Rocky 부팅 시 오류

```bash
...
[    2.456556] ---[ end Kernel panic - not syncing: Attempted to kill init! exitcode=0x00007f00 ]---
```

위와 같은 오류 발생 시, Rocky Linux 9가 요구하는 CPU 기능을 KVM 기본 설정이 충족하지 못해서 발생하는 것입니다.

이때는 기존의 KVM을 제거하고, 호스트 PC의 CPU 기능을 그대로 VM에 전달하도록 `-cpu host` 옵션을 추가합니다.

```bash
sudo kvm -m 2048 -smp 2 -cpu host -hda rocky-9-base.qcow2 -net nic -net user,hostfwd=tcp::2222-:22 -nographic
```

## 4. VM 내부 설정 변경

OpenStack의 Horizon 콘솔 로그 확인 및 하이퍼바이저 통신을 위해 아래 설정이 모든 이미지에 적용되어야 합니다.

### 4.1 Serial Console 활성화 (OpenStack 필수)

OpenStack 대시보드에서 로그를 보려면 커널 파라미터 수정이 필요합니다.

**Ubuntu/Debian:**

```bash
# /etc/default/grub 파일 수정
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT=".*"/GRUB_CMDLINE_LINUX_DEFAULT="console=tty1 console=ttyS0,115200n8"/' /etc/default/grub

# 수정 내용 적용
update-grub
```

**RHEL/Rocky/CentOS:**

```bash
# /etc/default/grub 파일 수정
sed -i 's/GRUB_CMDLINE_LINUX=".*"/GRUB_CMDLINE_LINUX="crashkernel=auto console=tty1 console=ttyS0,115200n8"/' /etc/default/grub

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

## 5. 옵션: NVIDIA GPU 드라이버 설치

**GPU 버전 이미지**를 생성할 때만 이 단계를 수행하세요. CPU 전용 이미지는 이 단계를 건너뜁니다.

> **전제**: 인스턴스에 이미 NVIDIA GPU가 Passthrough 되어 있거나, 로컬에서 드라이버 빌드 환경이 갖춰져야 합니다.

**Ubuntu/Debian:**

```bash
# 1. Nouveau 블랙리스트 처리
cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF

# 2. 변경 사항 커널에 적용
update-initramfs -u

# 3. 빌드 의존성 패키지 설치
apt-get update
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
  --property hw_qemu_guest_agent=yes
```
