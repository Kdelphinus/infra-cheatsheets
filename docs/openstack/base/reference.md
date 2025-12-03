# 🗺️ OpenStack Master Reference Map

이 문서는 Kolla-Ansible 기반 OpenStack 환경의 서비스 목록과 트러블 슈팅 포인트를 정리한 것입니다.

## 1\. 🏗️ 기반 인프라 (Infrastructure Backbone)

이 서비스들이 죽으면 OpenStack 전체가 마비됩니다. **장애 시 가장 먼저 체크해야 할 0순위 대상**입니다.

| 서비스 | 컨테이너명 (키워드) | 역할 | 🚑 트러블 슈팅 포인트 |
| :--- | :--- | :--- | :--- |
| **RabbitMQ** | `rabbitmq` | **메시지 버스.** 모든 서비스 간의 통신 담당. | • 증상: 모든 API가 느리거나 타임아웃, 로그에 `RPC timeout` 발생.<br>• 조치: 클러스터 상태 확인, 파티션 분리 여부 확인. |
| **MariaDB** | `mariadb` | **통합 DB.** 모든 리소스의 상태 정보 저장. | • 증상: `DB Connection Error`, 서비스 시작 불가.<br>• 조치: Max connection 초과 여부, Galera 클러스터 동기화 상태 확인. |
| **Memcached** | `memcached` | **캐시.** 인증 토큰, 콘솔 세션 캐싱. | • 증상: 로그인 느림, 콘솔 접속 실패.<br>• 조치: 서비스 재시작 (데이터 날아가도 무방). |
| **Valkey** | `valkey-*` | **KVS.** (구 Redis) 분산 락 관리 등. | • 증상: 특정 작업(Consoleauth 등) 실패. |

-----

## 2\. 🧠 컴퓨트 및 제어 (Compute & Control)

가상머신(VM)의 생애 주기를 관리합니다. **"VM이 안 만들어져요"** 할 때 보는 곳입니다.

| 서비스 | 컨테이너명 (키워드) | 역할 | 🚑 트러블 슈팅 포인트 |
| :--- | :--- | :--- | :--- |
| **Nova API** | `nova-api` | **명령 수신.** 사용자 요청을 받음. | • 증상: `openstack server create` 명령 실패 (500 Error). |
| **Nova Conductor** | `nova-conductor` | **중재자.** DB 접근 및 복잡한 로직 처리. | • 증상: VM 상태가 `Build`에서 멈추거나 `Error`로 빠짐. |
| **Nova Scheduler** | `nova-scheduler` | **배치.** VM을 어느 서버에 띄울지 결정. | • 증상: `No valid host was found` 에러 발생 (리소스 부족). |
| **Placement** | `placement-api` | **자원 관리.** CPU/RAM 재고 파악. | • 증상: 스케줄러 에러와 동반됨. 리소스 갱신 안됨. |
| **Ironic** *(Optional)* | `ironic-*` | **베어메탈.** 물리 서버 제어. | • 증상: 베어메탈 노드 배포 실패, PXE 부팅 실패. |
| **Zun** *(Optional)* | `zun-*` | **컨테이너.** VM 대신 컨테이너 직접 배포. | • 증상: 컨테이너 생성 실패. |

-----

## 3\. 🔌 네트워킹 (Networking)

**"VM은 떴는데 인터넷이 안 돼요"** 또는 **"IP가 안 받아져요"** 할 때 보는 곳입니다.

| 서비스 | 컨테이너명 (키워드) | 역할 | 🚑 트러블 슈팅 포인트 |
| :--- | :--- | :--- | :--- |
| **Neutron Server** | `neutron-server` | **API 서버.** 네트워크 설정 관리. | • 증상: 네트워크 생성 실패, 포트 할당 실패. |
| **Octavia** | `octavia-*` | **로드 밸런서(LB).** L4/L7 부하 분산. | • 증상: LB 생성 중 `Error`, Amphora(LB VM) 통신 불가. |
| **Designate** *(Optional)* | `designate-*` | **DNS.** 도메인 관리. | • 증상: DNS 레코드 전파 안됨. |
| **OVS / LinuxBridge** | *(Agent)* | **패킷 처리.** 실제 통신 담당. | • 증상: VM 통신 단절. (이건 컨테이너보다 호스트 네트워크 확인 필요) |

-----

## 4\. 💾 스토리지 (Storage)

데이터 저장소입니다. **"디스크 연결 에러"** 혹은 **"이미지 업로드 실패"** 시 확인합니다.

| 서비스 | 컨테이너명 (키워드) | 역할 | 🚑 트러블 슈팅 포인트 |
| :--- | :--- | :--- | :--- |
| **Cinder** | `cinder-*` | **블록 스토리지.** VM용 외장 디스크. | • 증상: 볼륨 생성 후 `Error`, Attach 실패 (iSCSI/Ceph 연결 문제). |
| **Glance** | `glance-api` | **이미지.** OS ISO/Image 저장. | • 증상: 이미지 업로드 중단, VM 생성 시 `Image not found`. |
| **Manila** | `manila-*` | **파일 공유.** NAS (NFS/CIFS). | • 증상: 공유 폴더 생성 실패, 마운트 타임아웃. |
| **Swift** *(Optional)* | `swift-*` | **오브젝트.** S3 호환 저장소. | • 증상: 파일 업로드/다운로드 실패, 복제(Replication) 지연. |
| **Ceph** *(Backend)* | `ceph-*` | **통합 스토리지 백엔드.** | • 증상: 전체 스토리지 서비스(Cinder, Glance, Swift) 동시 장애. |

-----

## 5\. 🛡️ 보안 및 인증 (Security & Auth)

**"로그인이 안 돼요"** 또는 **"권한이 없대요"** 할 때 봅니다.

| 서비스 | 컨테이너명 (키워드) | 역할 | 🚑 트러블 슈팅 포인트 |
| :--- | :--- | :--- | :--- |
| **Keystone** | `keystone` | **인증.** ID/PW 확인 및 토큰 발급. | • 증상: 모든 CLI 명령 시 `401 Unauthorized`. |
| **Barbican** *(Optional)* | `barbican-*` | **키 관리.** 암호화 키 저장. | • 증상: 암호화된 볼륨 생성 실패. |

-----

## 6\. 📊 운영 및 관리 (Operations)

시스템 상태를 보고 자동화하는 도구들입니다.

| 서비스 | 컨테이너명 (키워드) | 역할 | 🚑 트러블 슈팅 포인트 |
| :--- | :--- | :--- | :--- |
| **Horizon** | `horizon` | **웹 대시보드.** | • 증상: 웹 페이지 504 Gateway Time-out (주로 백엔드 API 문제). |
| **Heat** | `heat-*` | **오케스트레이션(IaC).** 스택 자동 배포. | • 증상: 스택 생성 실패 (상세 로그 확인 필수). |
| **Ceilometer** *(Optional)* | `ceilometer-*` | **데이터 수집.** 미터링. | • 증상: 사용량 데이터 누락. |

-----

## 🔧 빠른 트러블 슈팅 흐름도 (Cheat Sheet)

장애 발생 시 다음 순서로 로그를 확인하십시오.

1. **전체 시스템이 느리거나 이상하다?**
      * 👉 `RabbitMQ`, `MariaDB` 상태 먼저 확인 (가장 흔한 원인)
2. **CLI 명령어 자체가 안 먹힌다?**
      * 👉 `Keystone` 로그 확인 (`/var/log/kolla/keystone/keystone.log`)
3. **VM 생성(`server create`) 명령을 쳤는데 에러가 난다?**
      * 👉 `Nova-API` 로그 확인
4. **VM 상태가 `Build` → `Error`로 바뀐다?**
      * 👉 `Nova-Conductor` (로직 에러) 또는 `Nova-Compute` (하이퍼바이저 에러) 확인
      * 👉 `No valid host found` 에러라면 `Placement` 또는 `Nova-Scheduler` 확인
5. **VM은 떴는데 접속이 안 된다?**
      * 👉 `Neutron-Server` 로그 및 보안 그룹(Security Group) 확인
