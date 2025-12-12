# 3. 폐쇄망에서 Helm, Harbor, Ingress 설치

- 가이드 환경
  - OS: Rocky 9.6
  - kubelet: 1.30.14
- 폐쇄망용 K8s 설치 파일이 준비되어 있어야 합니다.

-----

## 🚀 Phase 1: Helm 설치 (Master-1 Only)

Helm은 마스터 노드에서 명령어를 내리는 도구이므로, **마스터 노드 1대**에만 설치하면 됩니다.

**[실행 위치: K8s-Master-Node-1]**

```bash
# 1. 바이너리 폴더로 이동
cd ~/offline-dist-split/k8s/binaries

# 2. 압축 해제 (이미 했다면 생략 가능)
tar -zxvf helm-v3.14.0-linux-amd64.tar.gz

# 3. 실행 파일을 시스템 경로로 이동
sudo mv linux-amd64/helm /usr/local/bin/helm

# 4. 설치 확인
helm version
# 결과: version.BuildInfo{Version:"v3.14.0", ...} 뜨면 성공
```

-----

## 🚀 Phase 2: 필수 인프라 설치 (Ingress & Storage)

Harbor가 정상 동작하려면 **외부 접속용 문(Ingress)**과 **데이터 저장소(StorageClass)**가 필수입니다.
이것들이 없으면 Harbor 파드가 뜨다가 `Pending` 상태로 멈춥니다.

**[실행 위치: K8s-Master-Node-1]**

### 1. Local Path Provisioner (저장소)

폐쇄망에서 가장 쉬운 스토리지 해결책입니다. 로컬 디스크 경로를 PV로 씁니다.

```bash
cd ~/offline-dist-split/k8s/utils

# 1. 설치
kubectl apply -f local-path-storage.yaml

# 2. (중요) 기본 스토리지 클래스로 지정
# 이걸 해야 Harbor가 "나 용량 줘" 할 때 자동으로 연결해줍니다.
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# 3. 확인 (local-path 옆에 (default)라고 떠야 함)
kubectl get sc
```

### 2. Ingress Nginx (접속 관문)

Harbor 웹 UI에 접속하기 위한 도구입니다.

아래 명령을 실행하여, worker 노드 중 하나를 선택합니다.

```bash
kubectl get node
```

Ingress Controller가 동작하는 노드를 고정합니다.

```bash
kubectl label node <NODE_NAME> ingress-ready=true
```

`ingress-nginx.yaml` 파일을 열어 `spec > template > spec` 아래 `nodeSelector` 부분을 추가합니다.

```bash
vi ingress-nginx.yaml

spec:
  # ... (생략)
  template:
    spec:
      ...
      nodeSelector:
        ingress-ready: "true"
      ...
```

```bash
# 1. 설치
kubectl apply -f ingress-nginx.yaml

# 2. 확인
# ingress-nginx-controller 파드가 Running 상태가 될 때까지 기다리세요.
watch kubectl get pods -n ingress-nginx
```

-----

## 🚀 Phase 3: Harbor 이미지 배포 (Worker Nodes)

**이게 제일 중요합니다.** Harbor 파드들은 **워커 노드**에 생성됩니다.
마스터에만 이미지가 있고 워커에 없으면, Harbor가 `ImagePullBackOff` 에러를 뿜으며 죽습니다.

**Bastion 서버**에서 Harbor 이미지를 워커 노드로 전송하고 로드했는지 체크하세요. (안 했다면 지금 해야 합니다\!)

**[실행 위치: Bastion 서버]**

```bash
# 1. 워커 노드로 Harbor 이미지 전송 (한 번도 안 보냈다면)
# (이미 보냈으면 생략 가능하지만, 확인차 다시 해도 됩니다)
WORKER_IPS=("10.10.10.73" "10.10.10.74" "10.10.10.75")
for IP in "${WORKER_IPS[@]}"; do
    echo "Sending K8s bundle to $IP..."
    scp dist-for-k8s-nodes.tar.gz rocky@$IP:~/
done
```

**[실행 위치: 각 Worker Node]**

```bash
# (각 워커 노드 접속 후)
tar -zxvf dist-for-k8s-nodes.tar.gz
cd ~/offline-dist-split/k8s/images

# Harbor 이미지 로드 (k8s.io 네임스페이스)
# Harbor 이미지는 'goharbor'로 시작합니다.
for img in goharbor*.tar; do
    echo "Loading $img..."
    sudo ctr -n k8s.io images import "$img"
done
```

-----

## 🚀 Phase 4: Harbor 설정 파일 작성 (values.yaml)

이제 마스터 노드로 돌아와서, Harbor 설치 설정을 만듭니다. **Notary(다운로드 실패한 기능)를 끄는 것**이 핵심입니다.

**용량과 노드를 꼭 확인하고 적절한 값으로 입력해야 합니다.**

**[실행 위치: K8s-Master-Node-1]**

```bash
# 작업 폴더 생성
mkdir -p ~/harbor-install
cd ~/harbor-install

# 설정 파일 생성
cat <<EOF > values-override.yaml
expose:
  type: ingress
  tls:
    enabled: false
  ingress:
    hosts:
      core: harbor.my.domain
      notary: notary.harbor.my.domain
    className: nginx

externalURL: http://harbor.my.domain

persistence:
  persistentVolumeClaim:
    registry:
      storageClass: "local-path"
      accessMode: ReadWriteOnce
      size: 100Gi  # 이미지 저장소
    jobservice:
      storageClass: "local-path"
      accessMode: ReadWriteOnce
      size: 10Gi   # 로그 및 작업 데이터 누적
    database:
      storageClass: "local-path"
      accessMode: ReadWriteOnce
      size: 20Gi   # 메타데이터 및 DB 안정성 확보
    redis:
      storageClass: "local-path"
      accessMode: ReadWriteOnce
      size: 10Gi   # 캐시 및 작업 큐 안정성
    trivy:
      storageClass: "local-path"
      accessMode: ReadWriteOnce
      size: 20Gi   # CVE DB 용량 확보

# [중요] 다운로드 실패한 Notary 기능 비활성화
notary:
  enabled: false

# --- 이미지 버전 강제 고정 (v2.10.0) ---
# 차트 기본값(v2.14.0) 대신 우리가 가진 v2.10.0을 사용

core:
  replicas: 1
  image:
    tag: "v2.10.0"
  nodeSelector:
    kubernetes.io/hostname: "k8s-worker-node-1"

jobservice:
  replicas: 1
  image:
    tag: "v2.10.0"
  nodeSelector:
    kubernetes.io/hostname: "k8s-worker-node-1"

portal:
  replicas: 1
  image:
    tag: "v2.10.0"
  nodeSelector:
    kubernetes.io/hostname: "k8s-worker-node-1"

registry:
  replicas: 1
  registry:
    image:
      tag: "v2.10.0"
  controller:
    image:
      tag: "v2.10.0"
  nodeSelector:
    kubernetes.io/hostname: "k8s-worker-node-1"

chartmuseum:
  enabled: false # 최신 Harbor에서는 잘 안 쓰지만 필요하면 true
  image:
    tag: "v2.10.0"
  nodeSelector:
    kubernetes.io/hostname: "k8s-worker-node-1"

# ------------------------------------------------------------
# 2. 데이터베이스 & Redis & Trivy (내부 컴포넌트)
# ------------------------------------------------------------

database:
  internal:
    image:
      tag: "v2.10.0"
    nodeSelector:
      kubernetes.io/hostname: "k8s-worker-node-1"

redis:
  internal:
    image:
      tag: "v2.10.0"
    nodeSelector:
      kubernetes.io/hostname: "k8s-worker-node-1"

trivy:
  enabled: true
  image:
    tag: "v2.10.0"
  nodeSelector:
    kubernetes.io/hostname: "k8s-worker-node-1"

exporter:
  image:
    tag: "v2.10.0"
  nodeSelector:
    kubernetes.io/hostname: "k8s-worker-node-1"
EOF
```

실제 도메인으로 바꿀 땐, 아래 명령어를 실행합니다.

```bash
sed -i 's/harbor.my.domain/새로운도메인주소/g' values-override.yaml
sed -i 's/k8s-worker-node-1/새로운노드이름/g' values-override.yaml
```

-----

## 🚀 Phase 5: Harbor 설치 (Helm Install)

**[실행 위치: K8s-Master-Node-1]**

charts 폴더에 있는 압축 폴더를 먼저 풀어준 뒤 진행합니다.

```bash
# 1. 차트 파일 위치 확인
# (아까 다운받은 harbor-*.tgz 파일이 있어야 합니다. 없으면 경로 확인!)
cd ~/offline-dist-split/k8s/charts/
tar -zxvf harbor-1.14.0.tgz

# 2. 설치 명령어 실행
helm install harbor harbor \
  -n harbor --create-namespace \
  -f ~/harbor-install/values-override.yaml
```

### 5.1 Harbor Core에서 i/o timeout 발생 시

**증상:**

- `JobService`, `Core` 등에서
  `dial tcp: lookup ... i/o timeout` 또는 `connection refused` 발생.
- CoreDNS는 Running 상태이나 DNS 조회(`nslookup`)가 타임아웃됨.
- 보안 그룹을 다 열어도 통신이 안 됨.

**원인:**

- 오픈스택(VXLAN) + Calico(IPIP) 이중 터널링으로 인한 패킷 오버헤드 발생.
- 기본 MTU(1440~1450) 설정 시 패킷이 잘려서(Fragment/Drop) DNS(UDP) 통신 불가.
- 보안 그룹에서 IPIP 프로토콜(Protocol 4) 차단 가능성.

**해결책 (필수 적용):**

1. **Calico MTU 강제 축소:**

- `kubectl edit configmap -n kube-system calico-config`
- `veth_mtu: "0"` (자동) → **`veth_mtu: "1350"`** (수동 고정)

2.**터널링 모드 변경 (IPIP → VXLAN):**

- `kubectl edit ippool default-ipv4-ippool`
- `ipipMode: Never`, `vxlanMode: Always` 로 변경.

3.**방화벽 해제:** 워커 노드 `firewalld` 비활성화 확인.

-----

## 🚀 Phase 6: 접속 테스트 (PC 설정)

Harbor는 도메인 기반으로 동작하므로, 접속하려는 \*\*내 PC(또는 Bastion)\*\*의 `hosts` 파일을 수정해야 들어갈 수 있습니다.

1. **Ingress 접속 IP 확인:**

    ```bash
    kubectl get ing -n harbor
    ```

      - `ADDRESS` 란에 IP가 나오면 그 IP입니다.
      - 만약 IP가 안 나오면, 워커 노드 중 \*\*아무 노드의 IP(예: 20.0.0.73)\*\*를 쓰면 됩니다.
      - 노드에 Floating IP가 적용되어있다면 해당 Floating IP를 사용해야 합니다.

2. **내 PC의 `/etc/hosts` (또는 윈도우 `C:\Windows\System32\drivers\etc\hosts`) 수정:**

    ```text
    # 예시 (워커 노드 IP가 20.0.0.73 이라고 가정)
    20.0.0.73  harbor.my.domain
    ```

3. **웹 브라우저 접속:**

    - 주소: `http://harbor.my.domain` (도메인을 변경했다면 변경한 도메인으로 접속해야 합니다.)
    - 기본 계정: `admin`
    - 기본 비번: `Harbor12345`

4. **이미지 업로드 및 다운로드**

    ```bash
    # 컨테이너디에 이미지 등록
    sudo ctr -n k8s.io images import <IMAGE>

    # 등록된 이미지 명 확인
    sudo ctr -n k8s.io images list | grep <IMAGE_NAME>

    # harbor 경로에 맞춰 이미지 이름 수정
    sudo ctr -n k8s.io images tag <CTR_IMAGE_NAME> harbor.my.domain/<HARBOR_PROJECT>/<IMAGE_NAMAE>

    # harbor에 등록
    # 현재 http 방식으로 띄었으므로 인증서 불필요
    sudo ctr -n k8s.io images push --plain-http -u admin:Harbor12345 harbor.my.domain/<HARBOR_PROJECT>/<IMAGE_NAME>

    # local 이미지 삭제
    sudo ctr -n k8s.io images remove harbor.my.domain/<HARBOR_PROJECT>/<IMAGE_NAME>

    # harbor 이미지 다운로드
    sudo ctr -n k8s.io images pull \
    --plain-http \
    -u admin:Harbor12345 \
    harbor.my.domain/<HARBOR_PROJECT>/<IMAGE_NAME>

    # 이미지 확인
    sudo ctr -n k8s.io images list | grep <IMAGE_NAME>
    ```
