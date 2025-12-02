# OpenStack K8s 네트워크 통신 장애 및 Harbor 배포 실패

## 1\. 개요 (Issue Summary)

- **증상:**

1. OpenStack 기반 Kubernetes 클러스터 구축 후, Pod 간 통신 및 Service IP(`ClusterIP`) 통신 불가.
2. `nslookup` 타임아웃 및 Pod 내부에서 외부 인터넷 통신 불가.
3. 네트워크 장애로 인해 Harbor 설치가 중단되었으며, `harbor-jobservice`가 `harbor-core`에 연결하지 못하는 오류(`connection refused`) 발생

- **영향도:** 클러스터 내 모든 애플리케이션 통신 두절, 신규 솔루션(Harbor) 배포 불가.

## 2\. 환경 정보 (Environment)

- **Infra:** OpenStack Flamingo (2025.2)
- **OS:** Ubuntu 22.04 LTS
- **Kubernetes:** v1.33 (Master 1, Worker 3)
- **CNI:** Calico (Overlay Network)
- **Application:** Harbor Registry

## 3\. 원인 분석 (Root Cause Analysis)

### 3.1. 핵심 원인: OpenStack Port Security

- **현상:** Kubernetes CNI(Calico)는 Pod마다 고유 IP 대역(예: `10.244.x.x`)을 할당함.
- **원인:** OpenStack Neutron의 **Port Security(포트 보안)** 기능은 VM의 네트워크 인터페이스(Port)에 할당된 고정 IP(`Fixed IP`) 이외의 트래픽이 감지되면 이를 \*\*IP 스푸핑(IP Spoofing) 공격으로 간주하여 차단(Drop)\*\*함.
- **결과:** Pod에서 나가는 모든 오버레이 트래픽이 OpenStack 네트워크 레벨에서 차단됨.

### 3.2. 부차적 혼란: ClusterIP의 특성

- **현상:** 네트워크 조치 후에도 Service IP(`10.96.0.1`)로 Ping이 되지 않아 해결되지 않은 것으로 오인.
- **원인:** K8s의 ClusterIP는 가상 IP(iptables/IPVS 규칙)이며, 기본적으로 **ICMP(Ping) 프로토콜을 처리하지 않음.**
- **결과:** Ping 실패는 정상이지만, 이를 네트워크 장애로 오판할 뻔함. (TCP/UDP 연결 테스트로 검증 필요)

### 3.3. Harbor 설치 실패 원인

- 초기 네트워크 단절 상태에서 설치를 시도하여 초기화 Job이 실패함.
- Helm 차트 설치 시 네임스페이스 미지정으로 인해 PVC가 `default` 네임스페이스에 생성되어, 재설치 시에도 기존 오염된 볼륨을 참조하는 문제 발생.

## 4\. 해결 과정 (Troubleshooting Steps)

### 4.1. 네트워크 진단

1. **테스트 Pod 배포:** `dns-test` 및 `netshoot` Pod 배포.
2. **증상 확인:**
    - `ping 10.96.0.1` (API Server) -\> **실패** (정상 동작임)
    - `nslookup kubernetes.default` -\> **실패** (Timeout, 비정상)
    - `wget https://10.96.0.1:443` -\> **실패** (Timeout, 비정상)
3. **조치 수행 (OpenStack):** 마스터 및 워커 노드의 OpenStack Port에 설정된 **Port Security 비활성화** 및 **Security Group 제거**.
4. **검증:**
    - `nslookup kubernetes.default` -\> **성공** (DNS 응답 수신)
    - `wget https://10.96.0.1:443` -\> **TLS Handshake Error** 발생 (네트워크 연결 성공 의미)
    - `nc -zv 10.96.0.1 443` -\> **Open** 확인.

### 4.2. Harbor 복구

1. **기존 설치 제거:** `helm uninstall harbor -n harbor`
2. **잔존 데이터 삭제 (중요):** `default` 네임스페이스에 잘못 생성된 PVC 일괄 삭제.
   - `kubectl delete pvc --all -n default` (Harbor 관련)
3. **재설치:** 네트워크 정상화 확인 후 Helm 재배포.

## 5\. 최종 해결책 (Solution)

### 5.1. OpenStack 네트워크 설정 변경

K8s 노드(VM)가 사용하는 Port에 대해 아래 두 가지 방법 중 하나 적용.

**방법 A: Port Security 비활성화 (빠른 해결)**
보안 그룹을 제거하고 포트 보안을 끔으로써 모든 트래픽 허용.

```bash
openstack port set --no-security-group --disable-port-security <PORT_ID>
```

**방법 B: Allowed Address Pairs 설정 (권장, 보안 강화)**
Port Security를 켜두되, Pod/Service 대역을 예외로 허용.

```bash
# 1. Pod 및 Service CIDR 허용
openstack port set --allowed-address ip_address=10.244.0.0/16 <PORT_ID>
openstack port set --allowed-address ip_address=10.96.0.0/12 <PORT_ID>

# 2. 보안 그룹에 IPIP/VXLAN/BGP 등 필수 프로토콜 허용 규칙 추가
# 3. Port Security 활성화
openstack port set --enable-port-security <PORT_ID>
```

### 5.2. 애플리케이션(Harbor) 재배포

네트워크가 복구된 후, 기존의 실패한 리소스(특히 PVC)를 완전히 정리하고 재설치하여 DB 초기화 프로세스가 정상적으로 수행되도록 함.

-----

## 6\. 교훈 및 참고 (Lessons Learned)

1. **OpenStack + K8s:** OpenStack 위에 K8s를 올릴 때는 반드시 **Port Security** 설정과 **Allowed Address Pairs**를 사전에 계획해야 한다.
2. **ClusterIP Ping:** Kubernetes Service IP(`ClusterIP`)는 Ping이 안 되는 것이 정상이다. 네트워크 테스트는 반드시 `curl`, `wget`, `nc`, `nslookup`을 사용해야 한다.
3. **Clean Re-install:** Stateful한 애플리케이션(DB 포함) 설치 실패 시, 단순히 Pod만 재시작하지 말고 **PVC를 포함한 모든 데이터를 삭제** 후 재설치하는 것이 가장 빠르고 확실한 해결책이다.
