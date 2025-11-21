# OpenStack 골든 이미지(Golden Image) 생성 가이드

OpenStack 환경(Nova, Glance)에서 인스턴스 배포 시 즉시 사용 가능한 깨끗한 Cloud Image를 만들기 위한 절차입니다.

> **필수 전제 조건**:
>
> - 작업은 로컬 KVM/QEMU 환경이나 OpenStack 내 임시 인스턴스에서 수행합니다.
> - 모든 명령은 **root 권한**으로 실행합니다.

-----

## 📋 목차

1. [공통: 필수 설정 (패키지 & 콘솔)](#1-공통-필수-설정-패키지--콘솔)
2. [옵션: NVIDIA GPU 드라이버 설치](#2-옵션-nvidia-gpu-드라이버-설치)
3. [OS별 구성: Ubuntu / Debian](#3-os별-구성-ubuntu--debian)
4. [OS별 구성: RHEL / Rocky / CentOS](#4-os별-구성-rhel--rocky--centos)
5. [공통: 최종 정리 및 이미지 변환](#5-공통-최종-정리-및-이미지-변환)

-----

## 1. 공통: 필수 설정 (패키지 & 콘솔)

OpenStack의 Horizon 콘솔 로그 확인 및 하이퍼바이저 통신을 위해 아래 설정이 모든 이미지에 적용되어야 합니다.

### 1.1 Serial Console 활성화 (OpenStack 필수)

OpenStack 대시보드에서 로그를 보려면 커널 파라미터 수정이 필요합니다.

**Ubuntu/Debian:**

```bash
# /etc/default/grub 파일 수정
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT=".*"/GRUB_CMDLINE_LINUX_DEFAULT="console=tty1 console=ttyS0,115200n8"/' /etc/default/grub
update-grub
```

**RHEL/Rocky/CentOS:**

```bash
# /etc/default/grub 파일 수정
sed -i 's/GRUB_CMDLINE_LINUX=".*"/GRUB_CMDLINE_LINUX="crashkernel=auto console=tty1 console=ttyS0,115200n8"/' /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 1.2 기본 패키지 설치

VMware Tools 대신 **QEMU Guest Agent**를 사용합니다.

* **Ubuntu**: `apt install -y qemu-guest-agent cloud-init`
* **Rocky/RHEL**: `yum install -y qemu-guest-agent cloud-init`

-----

## 2. 옵션: NVIDIA GPU 드라이버 설치

**GPU 버전 이미지**를 생성할 때만 이 단계를 수행하세요. CPU 전용 이미지는 이 단계를 건너뜁니다.

> **전제**: 인스턴스에 이미 NVIDIA GPU가 Passthrough 되어 있거나, 로컬에서 드라이버 빌드 환경이 갖춰져야 합니다.

### ✅ 설치 절차

1. **Nouveau 드라이버 비활성화 (필수)**
2. **빌드 의존성 설치**
3. **드라이버 설치 (CUDA Toolkit 포함 권장)**
4. **Persistence Mode 활성화**

### 💻 실행 명령어 (공통)

```bash
# 1. Nouveau 블랙리스트 처리
cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF
dracut --force  # 또는 update-initramfs -u (Ubuntu)

# 2. 의존성 설치
# Ubuntu
apt-get install -y build-essential linux-headers-$(uname -r) pkg-config libglvnd-dev
# Rocky/RHEL
yum groupinstall -y "Development Tools"
yum install -y kernel-devel-$(uname -r) kernel-headers-$(uname -r) epel-release dkms

# 3. NVIDIA 드라이버 설치 (Runfile 방식 권장 - 버전 제어 용이)
# (미리 다운로드 받은 .run 파일이 있다고 가정)
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/535.129.03/NVIDIA-Linux-x86_64-535.129.03.run
chmod +x NVIDIA-Linux-x86_64-*.run
./NVIDIA-Linux-x86_64-*.run --silent --dkms

# 4. Persistence Daemon 활성화 (부팅 시 로드 속도 향상)
nvidia-smi -pm 1
systemctl enable nvidia-persistenced
```

-----

## 3. OS별 구성: Ubuntu / Debian

> **대상**: Ubuntu 20.04, 22.04, 24.04 LTS

### ✅ 작업 순서

1. DataSource를 OpenStack으로 고정
2. SSH 설정
3. 네트워크/식별자 초기화 (VMware 가이드 참조하여 수정)

### 💻 실행 명령어

```bash
# 1. DataSource를 OpenStack으로 설정
# VMware와 달리 OpenStack을 최우선으로 둡니다.
cat > /etc/cloud/cloud.cfg.d/90_datasource.cfg << 'EOF'
datasource_list: [ OpenStack, ConfigDrive, Ec2, None ]
EOF

# 2. SSH 설정 (root 로그인 허용은 보안 정책에 따라 결정, 여기선 키 기반 접속 권장)
# 클라우드 이미지는 보통 cloud-user를 사용하므로 root 직접 접속은 막는 것이 표준입니다.
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart ssh

# 3. 네트워크 및 식별자 초기화
# (VMware 가이드의 정리 로직을 OpenStack용으로 적용)
rm -f /etc/netplan/*.yaml
rm -f /etc/network/interfaces.d/*
rm -f /etc/udev/rules.d/70-persistent-net.rules
rm -f /etc/ssh/ssh_host_*key*
truncate -s 0 /etc/machine-id

# 클라우드 설정 초기화
rm -rf /var/lib/cloud/*
rm -rf /var/log/cloud-init*

# 패키지 정리
apt-get clean
rm -rf /var/lib/apt/lists/*
```

-----

## 4. OS별 구성: RHEL / Rocky / CentOS

> **대상**: Rocky Linux 8/9, CentOS Stream 8/9

### ✅ 작업 순서

1. DataSource 설정
2. SELinux 설정 (필요시 Permissive)
3. 네트워크 설정 파일 삭제

### 💻 실행 명령어

```bash
# 1. DataSource 설정
cat > /etc/cloud/cloud.cfg.d/90_datasource.cfg << 'EOF'
datasource_list: [ OpenStack, ConfigDrive, Ec2, None ]
EOF

# 2. 네트워크 매니저 설정 정리
# (VMware 가이드의 정리 로직 적용)
rm -f /etc/sysconfig/network-scripts/ifcfg-e*
rm -f /etc/NetworkManager/system-connections/*.nmconnection

# 3. 식별자 초기화
rm -f /etc/udev/rules.d/70-persistent-net.rules
rm -f /etc/ssh/ssh_host_*key*
truncate -s 0 /etc/machine-id

# 4. 히스토리 및 로그 정리
history -c
rm -rf /var/lib/cloud/*
yum clean all
```

-----

## 5\. 공통: 최종 정리 및 이미지 변환

모든 설정이 끝났으면 마지막으로 cloud-init을 초기화하고 VM을 종료합니다.

### 5.1 최종 명령어

```bash
# 로그 파일 비우기
find /var/log -type f -exec truncate -s 0 {} \;

# cloud-init 정리 (가장 중요)
cloud-init clean --logs

# 시스템 종료 (재부팅 하지 마세요!)
shutdown -h now
```

### 5.2 Glance 이미지 업로드 (OpenStack CLI)

VM 이미지 파일(qcow2 등)을 OpenStack Glance에 업로드합니다.

**CPU 전용 이미지:**

```bash
openstack image create "Ubuntu-22.04-Base" \
  --file ubuntu-22.04-base.qcow2 \
  --disk-format qcow2 \
  --container-format bare \
  --public
```

**GPU 포함 이미지 (메타데이터 중요):**
OpenStack Nova가 GPU를 적절히 스케줄링하도록 메타데이터를 추가할 수 있습니다.

```bash
openstack image create "Rocky-9-NVIDIA-GPU" \
  --file rocky-9-gpu.qcow2 \
  --disk-format qcow2 \
  --container-format bare \
  --property hw_video_model=vga \
  --property hw_qemu_guest_agent=yes \
  --public
```

-----

### 💡 DevOps 엔지니어의 조언

1. **이미지 용량 최적화**: `qemu-img convert -O qcow2 -c input.qcow2 output.qcow2` 명령어를 사용하여 최종 이미지를 압축(`-c`)하면 스토리지 효율이 좋아집니다.
2. **CI/CD 파이프라인**: 이 과정을 수동으로 하기보다 **Packer**를 사용하여 '코드로서의 이미지(Image as Code)'를 구현하는 것을 강력히 권장합니다.
      * Packer OpenStack Builder를 사용하면 위 스크립트를 프로비저너로 실행하여 자동화할 수 있습니다.
