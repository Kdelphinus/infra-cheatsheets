# 🖥️ GPU Passthrough를 위한 OS 사전 설정 가이드

이 가이드는 OpenStack Compute 노드에서 GPU를 가상 머신(VM)에 직접 할당(Passthrough)하기 위해 필요한 **커널 및 드라이버 설정** 절차입니다.

## 📝 변수 정의 (치환 필요)

- `<VENDOR_ID>` : GPU 제조사 ID (예: NVIDIA의 경우 `10de`)
- `<PRODUCT_ID>` : GPU 모델 ID (예: `20b7`)

-----

## 1단계: IOMMU 활성화 (Grub 설정)

CPU의 가상화 기능을 활성화하고, 호스트 OS가 GPU를 점유하지 않도록 설정합니다.

### 1. 설정 파일 수정 (공통)

`/etc/default/grub` 파일을 열어 `GRUB_CMDLINE_LINUX` 라인을 수정합니다.

```bash
# 인텔 CPU 기준 설정 (AMD의 경우 intel_iommu 대신 amd_iommu=on 사용)
# 기존 라인 뒤에 아래 옵션들을 추가합니다.
# nouveau, nvidia 드라이버가 로드되지 않도록 blacklist 처리합니다.
GRUB_CMDLINE_LINUX="... intel_iommu=on iommu=pt modprobe.blacklist=nouveau,nvidia,nvidia_drm"
```

### 2. Grub 적용 및 재부팅 (OS별 선택)

**🅰️ Ubuntu 22.04:**

```bash
sudo update-grub
sudo reboot
```

**🅱️ Rocky Linux:**

```bash
# BIOS 모드일 경우
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
# UEFI 모드일 경우 (대부분의 최신 서버)
sudo grub2-mkconfig -o /boot/efi/EFI/rocky/grub.cfg

sudo reboot
```

### 3. IOMMU 활성화 확인

재부팅 후 설정이 적용되었는지 확인합니다.

```bash
cat /proc/cmdline
# 출력 결과에 intel_iommu=on 등이 포함되어 있어야 함
```

-----

## 2단계: GPU 장치 ID 확인 및 변수 확보

시스템에 장착된 GPU의 PCI ID를 식별합니다.

```bash
# NVIDIA GPU 검색 예시
lspci -nn | grep -i NVIDIA
```

**출력 예시:**
`03:00.0 3D controller [0302]: NVIDIA Corporation ... [10de:20b7] ...`

- 위 예시에서 `<VENDOR_ID>`는 `10de`, `<PRODUCT_ID>`는 `20b7`입니다.

-----

## 3단계: VFIO 모듈 로드 설정

GPU를 호스트 OS 대신 VM이 사용할 수 있도록 VFIO 드라이버를 로드합니다.

### 1. 수동 로드 (테스트용)

```bash
sudo modprobe vfio
sudo modprobe vfio_iommu_type1
sudo modprobe vfio-pci
```

### 2. 영구 적용 설정 (OS별 선택)

부팅 시 자동으로 모듈이 로드되도록 설정합니다.

**🅰️ Ubuntu 22.04:**

```bash
# /etc/modules 파일 편집
sudo vi /etc/modules

# 아래 내용 추가
vfio
vfio_iommu_type1
vfio_pci
```

**🅱️ Rocky Linux:**

```bash
# /etc/modules-load.d/vfio.conf 파일 생성
sudo vi /etc/modules-load.d/vfio.conf

# 아래 내용 추가
vfio
vfio_iommu_type1
vfio-pci
```

-----

## 4단계: GPU 장치 바인딩 (Driver Binding)

식별한 GPU ID를 VFIO 드라이버에 바인딩하여 호스트가 사용하지 못하게 합니다.

### 1. 바인딩 설정 파일 생성 (공통)

```bash
# /etc/modprobe.d/vfio.conf 파일 생성 또는 수정
sudo vi /etc/modprobe.d/vfio.conf

# 아래 내용 추가 (변수 치환 필요)
options vfio-pci ids=<VENDOR_ID>:<PRODUCT_ID>
```

### 2. 램디스크(Ramdisk) 업데이트 및 재부팅 (OS별 선택)

설정 내용을 초기 부팅 이미지에 반영합니다.

**🅰️ Ubuntu 22.04:**

```bash
sudo update-initramfs -u
sudo reboot
```

**🅱️ Rocky Linux:**

```bash
sudo dracut -f
sudo reboot
```

-----

## 5단계: 최종 검증

재부팅 후 GPU가 `vfio-pci` 드라이버를 사용 중인지 확인합니다.

### 1. 드라이버 확인

```bash
# Kernel driver in use가 vfio-pci로 되어 있어야 성공
lspci -nnk -d <VENDOR_ID>:<PRODUCT_ID>
```

### 2. (트러블슈팅) 바인딩이 안 된 경우 강제 적용

만약 위 단계 후에도 드라이버가 잡히지 않는다면 아래 명령어로 강제 바인딩을 시도합니다.

```bash
echo "<VENDOR_ID> <PRODUCT_ID>" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
```

이후 다시 `lspci -nnk` 명령어로 확인합니다.

-----

## RTX 계열의 GPU 사용 시

RTX 계열과 같은 소비자용 GPU는 가상화 환경을 고려하지 않기에 FLR(Function Level Reset) 기능을 지원하지 않습니다.
따라서 GPU 인스턴스를 종료하거나 재부팅하면, GPU 하드웨어가 초기화되지 않고 Stuck 상태로 남아 데이터 찌꺼기를 가지고 있다가 VM의 다음 동작을 방해하고, 이를 호스트가 응답이 없는 것으로 인식하고 인스턴스를 PAUSED 상태로 강제 정지시켜버립니다.

이를 막기 위해 `vendor-reset` 등의 프로그램을 사용하거나 임시로 쉘 스크립트를 실행하는 방법이 있습니다.
아래는 쉘 스크립트로 FLR 기능을 임시 조치하는 방법입니다.
모든 명령은 GPU 호스트 노드에서 실행해야 합니다.

```bash
cat << 'EOF' > qemu-hook.sh
#!/bin/bash
# GPU reset hook for RTX Passthrough VMs

VM_NAME="$1"
ACTION="$2"
PHASE="$3"

# 실제 ID는 lspci로 확인 후 수정 필요
GPU_PCI="0000:01:00.0"
AUDIO_PCI="0000:01:00.1" # 오디오 장치가 있다면 같이 리셋해야 안전함

# 로그 파일 위치 (Kolla 로그 폴더로 지정하여 호스트에서도 볼 수 있게 함)
LOGFILE="/var/log/libvirt/gpu-reset.log"

if [[ "$ACTION" == "stopped" && "$PHASE" == "end" ]]; then
    echo "[$(date)] VM $VM_NAME stopped. Resetting GPU..." >> $LOGFILE

    # 1. Remove (장치 제거)
    echo 1 > /sys/bus/pci/devices/$GPU_PCI/remove
    echo 1 > /sys/bus/pci/devices/$AUDIO_PCI/remove 2>/dev/null
    
    # 잠시 대기
    sleep 2

    # 2. Rescan (장치 재인식)
    echo 1 > /sys/bus/pci/rescan
    
    echo "[$(date)] GPU Reset done." >> $LOGFILE
fi
EOF
```

위 쉘 스크립트를 컨테이너 내부에 복사하고, 권한을 부여합니다.

```bash
# 1. 컨테이너 내부 디렉토리 생성
sudo docker exec -u root nova_libvirt mkdir -p /etc/libvirt/hooks

# 2. 파일 복사 (docker cp 사용)
sudo docker cp qemu-hook.sh nova_libvirt:/etc/libvirt/hooks/qemu

# 3. 권한 설정 (실행 권한 필수)
sudo docker exec -u root nova_libvirt chmod +x /etc/libvirt/hooks/qemu
sudo docker exec -u root nova_libvirt chown root:root /etc/libvirt/hooks/qemu

# 4. 컨테이너 재시작
sudo docker restart nova_libvirt
```

설정이 잘 적용되었다면, gpu 인스턴스가 삭제되었을 때마다 아래 로그가 찍히는 것을 확인할 수 있습니다.

```bash
sudo docker exec -it nova_libvirt cat /var/log/libvirt/gpu-reset.log
```

```bash
[Wed Nov 26 09:44:01 KST 2025] VM instance-0000001a stopped. Resetting GPU...
[Wed Nov 26 09:44:03 KST 2025] GPU Reset done.
[Wed Nov 26 09:49:32 KST 2025] VM instance-0000001b stopped. Resetting GPU...
[Wed Nov 26 09:49:34 KST 2025] GPU Reset done.
[Wed Nov 26 09:51:46 KST 2025] VM instance-0000001b stopped. Resetting GPU...
[Wed Nov 26 09:51:49 KST 2025] GPU Reset done.
[Wed Nov 26 09:56:32 KST 2025] VM instance-0000001c stopped. Resetting GPU...
[Wed Nov 26 09:56:35 KST 2025] GPU Reset done.
[Wed Nov 26 10:02:11 KST 2025] VM instance-0000001d stopped. Resetting GPU...
[Wed Nov 26 10:02:13 KST 2025] GPU Reset done.
[Wed Nov 26 10:03:10 KST 2025] VM instance-0000001d stopped. Resetting GPU...
[Wed Nov 26 10:03:12 KST 2025] GPU Reset done.
```
