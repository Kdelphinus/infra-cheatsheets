# PCI Passthrougu + Placement API 적용

## 주의 사항

- 이 문서는 아직 검증되지 않았습니다. 추후 검증 후, 이 항목을 제거할 예정입니다.

## 0. 사전 확인 사항

- Ubuntu 24.04 + Openstack 2025.2 환경에서 진행
- GPU 노드에 IOMMU 커널 설정 확인
- GPU 관리자가 VFIO인지 확인

## 1. Nova 설정 오버라이딩 (GPU 인식시키기)

Kolla-Ansible은 `/etc/kolla/config/nova/nova.conf` 파일을 우선순위로 적용합니다. 여기에 GPU 정보를 박제합니다.

```bash
# 사용할 PCI 장비의 이름 생성
# CUSTOM_ 접두사는 필수로 들어가야 함
openstack resource class create CUSTOM_<RESOURCE_NAME>
```

```bash
# 설정 디렉토리 생성
sudo mkdir -p /etc/kolla/config/nova

# 설정 파일 작성
sudo vi /etc/kolla/config/nova/nova.conf
```

<!-- end list -->

```ini
[pci]
# 1. [Compute 노드용] 물리 서버의 PCI 장치 화이트리스트
device_spec = [{"vendor_id": "<VENDOR_ID>", "product_id": "<PRODUCT_ID>"}]

# 2. [API 노드용] Flavor에서 사용할 별칭(Alias) 정의
# name은 나중에 Flavor 만들 때 씁니다. (여기서는 'nvidia-gpu'로 정함)
# nova가 장치를 못 찾겠다 할 때, type-PCI 변경 필요
alias = {"vendor_id": "<VENDOR_ID>", "product_id": "<PRODUCT_ID>", "device_type": "type-PCI", "name": "<RESOURCE_NAME>"}

# 3. PCI를 placement에 통합
report_in_placement = true

[filter_scheduler]
# 3. [Scheduler 노드용] PCI 필터 활성화 (기존 필터 뒤에 추가)
available_filters = nova.scheduler.filters.all_filters
enabled_filters = ComputeFilter,ImagePropertiesFilter,PciPassthroughFilter

# 존재하지 않는 옵션인지 확인 필요
pci_in_placement = true
```

### 1.1 2025.2 버전에 추가된 항목

#### GPU 보안 관련

`device_spec` 에서 `one_time_use` 항목을 추가 시, 사용자가 쓰고 반납한 GPU는 `reserved` 상태가 되어 다른 사용자가 사용할 수 없는 상태가 됩니다.

이를 관리자가 스크립트로 GPU 초기화 후, `openstack resource provider inventory set --reserved 0 ...` 명령을 실행해야 다시 사용할 수 있게 됩니다.

```ini
[pci]
device_spec = [{"vendor_id": "<VENDOR_ID>", "product_id": "<PRODUCT_ID>", "one_time_use": "true"}]
```

#### Live Migration

**live_migratable**은 **"GPU를 달고 있는 VM도 끄지 않고 다른 서버로 이사(Live Migration)갈 수 있게 해주는 스위치"**입니다.

이를 사용하기 위해선 아래와 같은 조건을 충족해야 합니다.

1. SR-IOV 사용 가능 여부
2. NVIDIA vGPU 라이선스

```ini
[pci]
device_spec = [{ "vendor_id": "<VENDOR_ID>", "product_id": "<PRODUCT_ID>", "live_migratable": "true" }]
alias = { "name": "<RESOURCE_NAME>", ... "live_migratable": "true" }
```

-----

## 2. 설정 적용 (Reconfigure)

기존 오픈스택을 끄지 않고 설정만 업데이트합니다.

```bash
# 가상환경 활성화 필수
source ~/kolla-venv/bin/activate

# 재설정 적용 (Nova 서비스들만 재시작됨)
kolla-ansible reconfigure -i multinode --tags nova
```

*(전체를 다 돌리면 오래 걸리니 `--tags nova`로 시간 단축 가능합니다.)*

-----

## 3. Placement 등록 검증

Nova가 GPU를 감지하고 Placement에 "나 GPU 있어\!"라고 보고했는지 확인합니다.

### 3.1. 물리 서버(Compute Node) UUID 확인

```bash
openstack resource provider list
```

- 여기서 GPU 노드의 자식 `ID` (UUID)를 복사하세요.

### 3.2. 인벤토리 조회

```bash
# [UUID] 자리에 복사한 ID를 넣으세요
openstack resource provider inventory list [UUID]
```

**[성공 판독]**
출력 결과에 아래와 비슷한 항목이 보여야 합니다.

- `PCI_DEVICE`: `total: 1` (또는 GPU 개수만큼)
- 또는 `CUSTOM_PCI_10DE_20B7`: `total: ...`

-----

## 4. 외부 네트워크(External Network) 뚫기

인스턴스가 외부(인터넷)와 통신하려면 **"물리 서버의 빈 깡통 인터페이스(예: eno2)"**와 연결된 가상 네트워크를 만들어야 합니다.

**전제 조건:**

- `globals.yml`에 `neutron_external_interface: "eno2"` (물리서버의 외부용 포트)가 설정되어 있어야 합니다.
- 해당 포트는 스위치에서 인터넷이 되는 VLAN(또는 Untagged)에 꽂혀 있어야 합니다.

### 4.1. 네트워크 생성 (Flat 타입)

사내망 IP 대역을 그대로 쓰는 방식입니다. (예: 사내망이 `192.168.0.0/24`라고 가정)

```bash
# 1. Provider Network 생성 (이름: public)
# physnet1은 kolla 기본 물리 네트워크 이름입니다.
openstack network create --share --external \
  --provider-physical-network physnet1 \
  --provider-network-type flat \
  public

# 2. Subnet 생성 (여기에 실제 사내망 정보를 넣어야 함!)
# gateway: 사내망 게이트웨이 (예: 192.168.0.1)
# allocation-pool: DHCP로 쓸 범위 (예: 100~200번)
openstack subnet create --network public \
  --allocation-pool start=192.168.0.100,end=192.168.0.200 \
  --dns-nameserver 8.8.8.8 \
  --gateway 192.168.0.1 \
  --subnet-range 192.168.0.0/24 \
  public_subnet
```

```bash
# 3. 네트워크 관련 설정(MTU 조절)
# VM 안에서 실행 (헤더 크기 감안해서 1450으로 핑)
ping -M do -s 1422 8.8.8.8  # (1422 + 28헤더 = 1450) -> 잘 될 겁니다.
ping -M do -s 1472 8.8.8.8  # (1472 + 28헤더 = 1500) -> 실패하거나 타임아웃 날 겁니다.

# 3-1. 1472 핑에서 오류 발생 시
# /etc/kolla/globals.yml
network_mtu: 1450 # (기본값 1500 -> 1450 축소)

# 3-2. 수정 후 재배포
kolla-ansible reconfigure -i multinode
```

> ### MTU 관련 속도 지연이 생기는 이유
>
> 1) 기본 패킷 크기 제한이 1500 byte, 오픈스택 내부에서는 패킷에 오버헤드(VXLAN, UDP, IP, Ethernet 헤더 -> 총 50 byte)를 붙여서 포장 후 전송  
> 2) 따라서 vm이 1500 byte 데이터를 전송 시, vm에서 50 byte추가되어 1550 byte 전송 요청  
> 3) 물리 랜카드에서 패킷 크기 초과 인지 -> 패킷 폐기 또는 쪼개기  
> 4) 패킷 손실 혹은 재전송이 발생하며 네트워크 속도 저하(특히 HTTPS 접속이나 대용량 다운로드 시)  

-----

## 5. 인스턴스 생성 테스트

이제 **"GPU가 달린 + 인터넷이 되는"** 인스턴스를 만듭니다.

### 5.1. GPU Flavor 생성

아까 `nova.conf`에서 정한 alias 이름(`nvidia-gpu`)을 씁니다.

```bash
openstack flavor create --ram 8192 --disk 50 --vcpus 4 gpu.flavor
openstack flavor set \
  --property "pci_passthrough:alias"="<RESOURCE_NAME>:1" \
  --property "resources:<CUSTOM_RESOURCE_CLASS>=1" \
  --property "hw:pci_numa_affinity_policy"="preferred" \
  --property hw_machine_type=q35 \
  gpu-flavor
```

> 만약 RTX 계열의 개인용 GPU를 사용한다면 `--property hw:kvm_hidden=true` 옵션도 함께 적용합니다.

> `pci_passthrough` 와 `resources` 를 같이 사용하면 개수가 이중 차감된다는 이슈가 보고된 것이 있습니다.
> 두 개를 모두 설정하는 것이 안정적이지만 2개가 같이 차감된다면 `resources` 속성을 제거해야 합니다.

### 5.2. 인스턴스 시작

```bash
openstack server create --flavor gpu.flavor \
  --image "Ubuntu 24.04" \
  --network internal \
  my-gpu-vm
```

### 5.3. 최종 확인

VM 접속 후:

- `lspci`: NVIDIA 장치가 보이는지?
- `ping 8.8.8.8`: 인터넷이 되는지?
- `nvidia-smi`: 드라이버 설치 후 GPU가 잡히는지?

## 6. API 기반 할당 검증 (Advanced Verification)

CLI나 대시보드 상에서는 확인하기 어려운 **"실제 리소스 점유 현황(Placement)"**과 **"Flavor 내부 설정(Nova)"**을 API를 통해 직접 검증합니다.

### 6.0. 환경 변수 설정

검증에 필요한 변수들을 미리 선언합니다. `< >`로 표시된 부분에 실제 값을 입력하세요.

```bash
# 1. 접속 정보
export CONTROLLER_IP=$(openstack endpoint list --service keystone --interface public -c URL -f value | awk -F/ '{print $3}' | cut -d: -f1)
export PLACEMENT_PORT=$(openstack endpoint list --service placement --interface public -c URL -f value | awk -F: '{print $3}' | cut -d/ -f1)
export NOVA_PORT=$(openstack endpoint list --service compute --interface public -c URL -f value | awk -F: '{print $3}' | cut -d/ -f1)

# 2. 검증 대상 UUID (사전 확인 필요)
# 생성된 인스턴스의 ID
export SERVER_ID="<INSTANCE_ID>"
# 생성한 GPU Flavor의 ID
export FLAVOR_ID="<FLAVOR_ID>"
# GPU 자원을 가진 물리 노드(혹은 자식 노드)의 UUID
export RP_UUID=$(openstack resource provider allocation show $SERVER_ID -c resource_provider -c resources -f value | grep "CUSTOM_" | awk '{print $1}')
```

### 6.1. 인증 토큰 발급 (토큰이 없는 경우)

```bash
# 관리자 권한 로드 후 토큰 발급
source /etc/kolla/admin-openrc.sh
export OS_TOKEN=$(openstack token issue -c id -f value)
echo "Token: $OS_TOKEN"
```

### 6.2. Placement API 검증 (하드웨어 관점)

GPU 자원이 물리적으로 존재하며, 생성한 인스턴스가 이를 실제로 점유하고 있는지 확인합니다.

#### 1) Resource Inventory 조회 (재고 확인)

```bash
curl -s -X GET "http://${CONTROLLER_IP}:${PLACEMENT_PORT}/resource_providers/${RP_UUID}/inventories" \
  -H "X-Auth-Token: ${OS_TOKEN}" \
  -H "OpenStack-API-Version: placement 1.28" \
  -H "Content-Type: application/json" | json_pp
```

> **[확인 포인트]**
>
> - `total` 값이 실제 물리 GPU 개수와 일치하는지 확인
> - 리소스 클래스 이름이 `CUSTOM_<RESOURCE_NAME>` (또는 자동 생성된 이름)인지 확인

#### 2) Resource Allocations 조회 (점유 확인 ★중요)

```bash
curl -s -X GET "http://${CONTROLLER_IP}:${PLACEMENT_PORT}/resource_providers/${RP_UUID}/allocations" \
  -H "X-Auth-Token: ${OS_TOKEN}" \
  -H "OpenStack-API-Version: placement 1.28" \
  -H "Content-Type: application/json" | json_pp
```

> **[확인 포인트]**
>
> - 응답 JSON의 Key 값(Consumer UUID)에 **`<INSTANCE_ID>`**가 포함되어 있어야 함
> - 해당 인스턴스가 `CUSTOM_<RESOURCE_NAME>` 자원을 `1`개 사용 중이어야 함

### 6.3. Nova API 검증 (소프트웨어 관점)

인스턴스가 올바른 Flavor와 스펙으로 생성되었는지 확인합니다.

#### 1) Flavor 상세 스펙(Extra Specs) 조회

Flavor 조회 시 기본 정보에는 GPU 관련 설정이 나오지 않으므로, **`/os-extra_specs`** 엔드포인트를 호출해야 합니다.

```bash
curl -s -X GET "http://${CONTROLLER_IP}:${NOVA_PORT}/v2.1/flavors/${FLAVOR_ID}/os-extra_specs" \
  -H "X-Auth-Token: ${OS_TOKEN}" \
  -H "Content-Type: application/json" | json_pp
```

> **[확인 포인트]**
>
> - `"pci_passthrough:alias": "<RESOURCE_NAME>:1"` 설정 존재 여부
> - `"resources:CUSTOM_<RESOURCE_CLASS>": "1"` 설정 존재 여부
> - `"hw:pci_numa_affinity_policy": "preferred"` 설정 존재 여부

#### 2) Instance 상세 정보 조회

```bash
curl -s -X GET "http://${CONTROLLER_IP}:${NOVA_PORT}/v2.1/servers/${SERVER_ID}" \
  -H "X-Auth-Token: ${OS_TOKEN}" \
  -H "Content-Type: application/json" | json_pp
```

> **[확인 포인트]**
>
> - `status`: **"ACTIVE"**
> - `flavor.id`: 위에서 조회한 `<FLAVOR_ID>`와 일치해야 함
