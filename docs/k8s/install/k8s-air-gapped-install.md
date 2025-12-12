# 2. 폐쇄망에서 Kubernetes와 Helm 설치

- 가이드 환경
  - OS: Rocky 9.6
  - kubelet: 1.30.14
- 폐쇄망용 K8s 설치 파일이 준비되어 있어야 합니다.

## 📦 Phase 0: 파일 배포 (Master -> Workers)

현재 마스터 노드(`Master`)에 `dist-for-k8s-nodes.tar.gz` 파일이 있다고 가정합니다.
설치를 위해 **워커 노드 3대에도 이 파일이 똑같이 있어야 합니다.**

**[실행 위치: K8s-Master-Node]**

```bash
# 워커 노드 IP 리스트 (환경에 맞게 수정하세요)
WORKER_IPS=("10.10.10.73" "10.10.10.74" "10.10.10.75")

# 반복문으로 파일 전송
for IP in "${WORKER_IPS[@]}"; do
    echo "Sending file to $IP..."
    scp ~/dist-for-k8s-nodes.tar.gz rocky@$IP:~/
done
```

> **Note:** 전송이 끝나면, \*\*모든 노드(Master 1대, Worker 3대)\*\*에서 압축을 풀어주세요.
>
> ```bash
> tar -zxvf ~/dist-for-k8s-nodes.tar.gz
> ```

-----

## 🚀 Phase 1: 로컬 패키지 설치 & OS 설정 (전체 노드)

**[실행 위치: Master 1, Worker 1~3 전체]**

Repo 설정(`yum.repos.d`)을 건드리지 않고, `dnf` 명령어로 다운받은 RPM 파일들을 직접 설치합니다.

### 1. RPM 파일 설치 (Local Install)

파일에 9.7 버전을 포함하고 있어서 이를 9.6으로 강제 설치하는 방법을 사용합니다.

```bash
# 압축 푼 디렉토리로 이동
cd ~/offline-dist-split

# 1. 기존 Repo 비활성화 후 로컬 RPM 일괄 설치
# -Uvh: Upgrade (설치 또는 업그레이드) + Verbose (상세표시) + Hash (진행바)
# --force: 이미 설치된 패키지라도 강제로 덮어쓰기
# --nodeps: 순환 의존성 오류 무시 (일단 다 설치해라)
sudo rpm -Uvh --force --nodeps common/rpms/*.rpm k8s/rpms/*.rpm

# 2. 설치 확인
rpm -qa | grep -E "kubelet|containerd|haproxy"
# 결과에 패키지 이름들이 뜨면 성공입니다.
```

### 2. 필수 OS 설정 (매우 중요)

```bash
# 1. Swap 비활성화
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# 2. SELinux Permissive 모드
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 3. 커널 모듈 로드
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# 4. 커널 파라미터 적용
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

### 3. Containerd 설정 및 실행

```bash
# 1. 기본 설정 생성
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# 2. SystemdCgroup 활성화 (필수!)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 3. pause 버전 변경(이전 설치 파일에서 3.9 버전으로 준비)
sudo sed -i 's/pause:3.10.1/pause:3.9/g' /etc/containerd/config.toml

# 3. 서비스 시작
sudo systemctl enable --now containerd
sudo systemctl enable --now kubelet
```

### 4. hosts 파일 설정

```bash
sudo vi /etc/hosts

<master node ip> <master node hostname>
<worker1 node ip> <worker1 node hostname>
<worker2 node ip> <worker2 node hostname>
<worker3 node ip> <worker3 node hostname>
```

-----

## 🚀 Phase 2: 이미지 로드 (전체 노드)

Repo가 없으니 `docker pull`은 불가능합니다. 가져온 `.tar` 파일을 `containerd`에 직접 집어넣습니다.

**[실행 위치: Master 1, Worker 1\~3 전체]**

```bash
# 이미지 폴더로 이동
cd ~/offline-dist-split/k8s/images

# 반복문으로 로드 (시간 소요됨)
# k8s.io 네임스페이스에 이미지를 등록합니다.
for img in *.tar; do
    echo "Loading $img..."
    sudo ctr -n k8s.io images import "$img"
done

# 확인
sudo ctr -n k8s.io images list | grep kube-apiserver
```

-----

## 🚀 Phase 3: 로드밸런서(LB) 구성 (Master가 1대라면 Phase 4로 넘어갑니다)

Master 노드 3대(`10.10.10.70`, `71`, `72`)와 **가상 IP(VIP, `10.10.10.200`)** 를 환경을 가정했습니다.

현재 **K8s-Master-Node-1**에서 이 작업을 수행하시면 됩니다.

이 설정은 **K8s API Server(6443 포트)** 앞단에 VIP를 두어, 마스터 노드가 죽어도 API 통신이 끊기지 않게 합니다.

### 1. 사전 커널 설정 (필수)

가상 IP(VIP)가 내 인터페이스에 없어도 바인딩할 수 있도록 커널 파라미터를 수정해야 합니다.
이 작업을 수행하지 않으면 HAProxy가 시작 실패할 수 있습니다.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/haproxy.conf
net.ipv4.ip_nonlocal_bind = 1
EOF

sudo sysctl --system
```

### 2. HAProxy 설정 (Load Balancer)

실제 마스터 노드 3대로 트래픽을 분산시키는 설정입니다.

- **파일:** `/etc/haproxy/haproxy.cfg`
- **수정:** 기존 내용을 백업하고 아래 내용으로 덮어쓰세요.
- **주의:** `server` 섹션의 IP들은 사용자 환경의 실제 마스터 IP로 맞춰주세요.

<!-- end list -->

```bash
# 기존 파일 백업
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

# 설정 파일 작성
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

defaults
    mode                    tcp
    log                     global
    option                  tcplog
    option                  dontlognull
    option                  redispatch
    retries                 3
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout check           10s
    maxconn                 3000

# ---------------------------------------------------------------------
# Kubernetes API Server LB
# ---------------------------------------------------------------------
frontend k8s-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-masters

backend k8s-masters
    mode tcp
    balance roundrobin
    option tcp-check
    
    # 3대 노드 등록 (현재 1대만 켜져 있어도 설정은 다 넣어둡니다)
    server master1 10.10.10.70:6443 check fall 3 rise 2
    server master2 10.10.10.71:6443 check fall 3 rise 2
    server master3 10.10.10.72:6443 check fall 3 rise 2
EOF
```

### 3. Keepalived 설정 (VIP 관리)

VIP(`10.10.10.200`)를 띄우는 설정입니다.

- **파일:** `/etc/keepalived/keepalived.conf`
- **확인할 것:** `interface` 부분이 `eth0`인지 `ens3`인지 `ip addr` 명령어로 꼭 확인하고 수정하세요\!

<!-- end list -->

```bash
# 설정 파일 작성
cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL
}

vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state MASTER            # Master-1은 MASTER, 나머지는 BACKUP
    interface eth0          # <--- 본인 네트워크 인터페이스명으로 변경 필수! (예: ens3)
    virtual_router_id 51
    priority 101            # Master-1은 101, 나머지는 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass 42        # 암호는 아무거나 (모든 노드 동일하게)
    }

    virtual_ipaddress {
        10.10.10.200        # 사용할 VIP (가상 IP)
    }

    track_script {
        check_haproxy
    }
}
EOF
```

### 4. 서비스 시작 및 검증

이제 두 서비스를 시작합니다.

```bash
# 서비스 시작
sudo systemctl enable --now haproxy
sudo systemctl enable --now keepalived

# 상태 확인
systemctl status haproxy keepalived
```

#### ✅ 성공 확인 방법 (매우 중요)

VIP가 제대로 떴는지 확인해봅니다.

```bash
ip addr show eth0  # (또는 설정한 인터페이스)
```

출력 결과에 `inet 10.10.10.200/32 ... secondary` 처럼 **VIP가 보이면 성공**입니다\!

-----

**Tip:** 현재 Master 2, 3번은 꺼져 있거나 K8s가 안 돌고 있습니다.
따라서 `HAProxy` 로그를 보면 master2, master3는 `DOWN` 상태라고 나올 겁니다.
정상이니 무시하고 **Master 1이 UP 상태**이고 **VIP가 핑이 되는지**만 확인하세요.

3중화 작업을 했다면 이후부턴 초기화할 때 이 할당한 VIP(예시에선 10.10.10.200)를 써야 합니다.

-----

## 🚀 Phase 4: 마스터 초기화 (Master-1 Only)

**[실행 위치: K8s-Master-Node-1]**

상황에 맞는 옵션 하나를 선택해서 실행하세요. 이때 `join` 명령어를 꼭 저장해두세요.

```bash
# 2. (중요) Join 명령어 저장
# 화면 맨 아래에 출력된 토큰 정보를 복사해서 메모장에 저장해두세요.
# HA 구성(옵션 A)을 했다면 명령어라 두 종류가 나옵니다.
# 1) Master Node Join용 (certificate-key 포함됨)
# 2) Worker Node Join용 (token, hash만 있음)
```

### 🅰️ 옵션 A: [추천] HA(3중화) 확장 대비 구성 (VIP 사용)

- **조건:** Phase 3에서 HAProxy와 VIP(`10.10.10.200`) 설정을 마친 경우.
- **장점:** 지금은 마스터가 1대여도, 나중에 명령어 한 줄로 마스터 2, 3번을 추가할 수 있습니다.
- **핵심:** `--control-plane-endpoint`에 VIP를 적고, `--upload-certs`를 추가합니다.

<!-- end list -->

```bash
# 1. 초기화 실행 (HA 모드)
# --upload-certs: 인증서를 K8s Secret에 올려서 다른 마스터가 쉽게 조인하게 함
sudo kubeadm init \
  --control-plane-endpoint "10.10.10.200:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16 \
  --kubernetes-version v1.30.0
```

### 🅱️ 옵션 B: 단순 1대 구성 (IP 직접 사용)

- **조건:** 로드밸런서 없이 그냥 혼자 쓸 경우.
- **단점:** 나중에 마스터를 늘리려면 인증서를 다시 생성하고 설정을 뜯어고쳐야 해서 매우 복잡합니다.

<!-- end list -->

```bash
# 1. 초기화 실행 (단일 모드)
# 엔드포인트에 본인 IP(10.10.10.70)를 넣거나 아예 생략 가능
sudo kubeadm init \
  --control-plane-endpoint "10.10.10.70:6443" \
  --pod-network-cidr=192.168.0.0/16 \
  --kubernetes-version v1.30.0
```

-----

### 📋 공통 후속 작업 (초기화 성공 후)

초기화가 성공적으로 끝나면 `Your Kubernetes control-plane has initialized successfully!` 가 뜹니다.
그 후 아래 작업을 수행하세요.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

-----

## 🚀 Phase 5: CNI (Calico) 설치

CNI 설치는 단일/다중 마스터 여부와 상관없이 **동일합니다.**
마스터 노드가 `NotReady` 상태에서 `Ready` 상태로 바뀌게 해줍니다.

**[실행 위치: K8s-Master-Node-1]**

```bash
# 1. Calico 매니페스트 위치로 이동
cd ~/offline-dist-split/k8s/utils

# 2. Calico 설치
kubectl apply -f calico.yaml

# 3. 설치 확인 (Watch 모드)
# 처음엔 NotReady였다가, calico-node 파드가 뜨면 Ready로 바뀝니다.
watch kubectl get nodes
```

만약 `calico` 가 배포되지 않는다면 kubelet에서 오류가 발생했을 확률이 높습니다.
이때는 아래 방법으로 초기화 후, `kubeadm init` 명령어를 재실행합니다.

```bash
# 1. 기존에 하다 만 설정 싹 지우기
sudo kubeadm reset -f

# 2. CNI 설정 파일 및 기존 인증서 잔여물 삭제 (충돌 방지)
sudo rm -rf /etc/cni/net.d
rm -rf $HOME/.kube/config
rm -rf /var/lib/etcd/
```

-----

## 🚀 Phase 6: 워커 노드 조인 (Worker 1~3 Only)

**[실행 위치: Worker 1, 2, 3]**

아까 복사해 둔 `kubeadm join` 명령어를 붙여넣으세요.

```bash
# 예시
sudo kubeadm join 10.10.10.70:6443 --token <토큰> \
    --discovery-token-ca-cert-hash sha256:<해시값>
```

만약 `[WARNING Hostname]: hostname "k8s-worker-node-1.novalocal" could not be reached`와
같은 오류가 발생한다면 `hosts` 파일에 해당 이름을 추가해야 합니다.

```ini
20.0.0.22 k8s-master-node
20.0.0.203 k8s-worker-node-1 k8s-worker-node-1.novalocal
20.0.0.222 k8s-worker-node-2 k8s-worker-node-2.novalocal
20.0.0.153 k8s-worker-node-3 k8s-worker-node-3.novalocal
```

### 💡 (참고) Worker Node Join 방법 차이

나중에 워커 노드를 붙일 때(`Phase 6`)도 구성에 따라 명령어가 약간 다릅니다. 미리 참고하세요.

- **HA 구성(옵션 A) 시:** 워커 노드는 **VIP(10.10.10.200)**를 바라보고 Join 합니다.

    ```bash
    kubeadm join 10.10.10.200:6443 --token <토큰> ...
    ```

- **단일 구성(옵션 B) 시:** 워커 노드는 **Master IP(10.10.10.70)**를 바라보고 Join 합니다.

    ```bash
    kubeadm join 10.10.10.70:6443 --token <토큰> ...
    ```

여기까지 완료되면 `kubectl get nodes` 명령어로 **Master 노드가 Ready 상태**인지 확인해 주세요.

-----

## ✅ 완료 확인

다시 마스터 노드에서:

```bash
kubectl get nodes
```

모두 `Ready` 상태라면 K8s가 정상 설치된 것입니다.
