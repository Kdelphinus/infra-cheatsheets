# 3. 폐쇄망에서 Helm, Harbor, Envoy(Ingress는 비권장) 설치

- 가이드 환경
  - OS: Rocky 9.6
  - kubelet: 1.30.14
- 폐쇄망용 K8s 설치 파일이 준비되어 있어야 합니다.
- [설치 파일 위치](https://drive.google.com/drive/folders/1joMQRpZPWzKgU9BBsdxy3b0qzJMWpBC8?usp=sharing)

-----

## 🚀 Phase 1: Helm 설치 (Master-1 Only)

Helm은 마스터 노드에서 명령어를 내리는 도구이므로, **마스터 노드 1대**에만 설치하면 됩니다.

**[실행 위치: K8s-Master-Node-1]**

```bash
# 1. 바이너리 폴더로 이동
cd ~/k8s-1.30/k8s/binaries

# 2. 압축 해제 (이미 했다면 생략 가능)
tar -zxvf helm-v3.14.0-linux-amd64.tar.gz

# 3. 실행 파일을 시스템 경로로 이동
sudo mv linux-amd64/helm /usr/local/bin/helm

# 4. 설치 확인
helm version
# 결과: version.BuildInfo{Version:"v3.14.0", ...} 뜨면 성공
```

-----

## 🚀 Phase 2: Envoy 설치

2026년 3월부터 `Ingress Nginx` 에 대한 공식 지원이 종료됩니다.
이에 따라 Kubernetes의 `Gateway API` 와 `Envoy` 를 사용하여 합니다.

### 0. 아키텍처 개요 (Standard Architecture)

쿠버네티스 보안 및 네트워크 표준을 준수하는 구성입니다.

- **Network:** `hostNetwork: false` (Pod는 K8s 내부망 사용, 노드 네트워크와 격리)
- **Service:** `type: LoadBalancer` (외부 트래픽 진입점)
- **Traffic Flow:**
`Client` -> `External IP (LB)` -> `Service (80/443)` -> `Envoy Pod (10080/10443)`
-> `Backend Pod`

### 1. 이미지 로드 (전체 노드)

**[실행 위치: Master 1, Worker 1~3 전체]**

전체 노드에 `envoy` 이미지들을 로드합니다.

```bash
cd ./envoy-1.36.3
sudo bash ./images/upload_images.sh
```

**[실행 위치: Master 1]**

마스터 노드로 돌아와 설치 스크립트를 실행합니다.

```bash
sudo bash install_envoy-gateway.sh
```

### 2. 배포 후 상태 확인 및 IP 할당

배포가 완료되면 가장 먼저 **Gateway Service의 External IP** 할당 상태를 확인해야 합니다.

```bash
# Envoy Gateway가 생성한 LoadBalancer 서비스 확인
kubectl get svc -n envoy-gateway-system | grep -i load
```

위 명령 실행 결과(`EXTERNAL-IP`)에 따라 조치 방법이 다릅니다.

1. **Case A: 클라우드 (AWS EKS, GKE, AKS 등)**
   - `EXTERNAL-IP`에 자동으로 IP 또는 도메인이 할당됩니다. **(별도 조치 불필요)**

2. **Case B: 온프레미 (MetalLB가 있는 경우)**
    - 설정된 IP Pool에서 자동으로 IP가 할당됩니다. **(별도 조치 불필요)**

3. **Case C: 온프레미 (MetalLB가 없는 경우) - `<pending>` 상태**
    - IP를 할당해 줄 컨트롤러가 없으므로 **수동으로 VIP(Node IP)를 바인딩**해야 합니다.

#### 🛠️ [Case C] 수동 IP 할당 명령어

서비스가 `<pending>` 상태로 멈춰 있을 때만 실행합니다.
`externalIPs` 에 있는 `1.1.1.213` 부분은 실제 사용할 노드 IP로 대체합니다.

```bash
# 1. IP가 할당되지 않은 서비스 이름 확인
SVC_NAME=$(kubectl get svc -n envoy-gateway-system -o jsonpath='{.items[?(@.spec.type=="LoadBalancer")].metadata.name}')

# 2. 실제 사용할 노드 IP로 패치 (IP 부분 수정 필수)
kubectl patch svc -n envoy-gateway-system $SVC_NAME \
  --type merge \
  -p '{"spec":{"externalIPs":["1.1.1.213"]}}'

echo "✅ 서비스($SVC_NAME)에 외부 IP(1.1.1.213)가 할당되었습니다."
```

### 3. 라우팅(HTTPRoute) 설정 및 검증

Gateway가 정상적으로 떴다면, 애플리케이션 연결 규칙(`HTTPRoute`)을 점검합니다.
이 검증은 서비스에 접근하지 못할 때 진행해도 됩니다.

> 서비스보다 먼저 envoy를 설치하기 때문에 현재는 생성된 HTTPRoute 자원이 없습니다.

#### ✅ 체크리스트 1: Gateway 이름 일치 여부

`HTTPRoute` 리소스가 현재 실행 중인 Gateway(`cmp-gateway`)를 정확히 가리키고 있어야 합니다.

```bash
# parentRefs가 'cmp-gateway'인지 확인
kubectl get httproute -A

```

**수정 방법:**

```bash
kubectl patch httproute <ROUTE_NAME> -n <NAMESPACE> --type='json' \
  -p='[{"op": "replace", "path": "/spec/parentRefs/0/name", "value": "cmp-gateway"}]'

```

#### ✅ 체크리스트 2: 백엔드 포트 (Connection Refused)

Envoy는 서비스(ClusterIP) 포트가 아닌 **파드(Pod)의 실제 컨테이너 포트**로 접속을 시도합니다.

- **증상:** 503 Service Unavailable 또는 Connection Refused
- **해결:** `HTTPRoute`의 `backendRefs` 포트를
**TargetPort(실제 앱 포트, 예: 8080)** 로 설정해야 합니다.

```bash
# 포트를 80 -> 8080으로 변경하는 예시
kubectl patch httproute <ROUTE_NAME> -n <NAMESPACE> --type='json' \
  -p='[{"op": "replace", "path": "/spec/rules/0/backendRefs/0/port", "value": 8080}]'
```

#### ✅ 체크리스트 3: 경로 재작성 (URL Rewrite)

애플리케이션이 하위 경로(Context Path)를 인식하지 못해 404가 발생할 경우 사용합니다.

- **상황:** `/oauth2/login` 호출 시 앱이 `/oauth2`를 경로로 인식하여 오류 발생.
- **해결:** `URLRewrite` 필터 적용.

```yaml

filters:
- type: URLRewrite
  urlRewrite:
    path:
      type: ReplacePrefixMatch
      replacePrefixMatch: /

```

### 4. 운영 및 로그 확인

Envoy Gateway는 동적으로 리소스를 관리하므로 파드 이름이 변경됩니다.
**Label Selector(`-l`)**를 사용하여 로그를 확인하는 것이 표준입니다.

#### 📋 프록시(Data Plane) 로그

실제 트래픽 처리, 접속 오류 확인 시 사용합니다.

```bash
# Envoy Proxy 로그 실시간 확인
kubectl logs -n envoy-gateway-system -f -l gateway.envoyproxy.io/owning-gateway-name=cmp-gateway

```

#### 🧠 컨트롤러(Control Plane) 로그

Gateway 설정 변환, 배포 실패 원인 분석 시 사용합니다.

```bash
# Gateway Controller 로그 확인
kubectl logs -n envoy-gateway-system -f -l app.kubernetes.io/name=envoy-gateway

```

### (예전)Ingress Nginx

> 2026년 3월부터 Ingress Nginx에 대한 지원이 종료됩니다.
> 이에 따라 Ingress Nginx 대신 위에 있는 Gateway API + Envoy를 사용을 적극 권장합니다.

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

kind: Deployment
...
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

만약 LB가 없어서 노드에 직접 붙어야 하는 상황이라면 ingress-nginx.yaml 파일에 hostNetwork 옵션을 추가해주세요.

```yaml
spec:
  template:
    spec:
      hostNetwork: true  # <--- 이 줄을 추가하세요! (dnsPolicy 근처에 두면 됩니다)
      dnsPolicy: ClusterFirst
      containers:
      - name: controller
        ...
```

-----

## 🚀 Phase 3: Harbor 설치

### 0. Local Path Provisioner (저장소, 선택)

**[실행 위치: K8s-Master-Node-1]**

폐쇄망에서 가장 쉬운 스토리지 해결책입니다. 로컬 디스크 경로를 PV로 씁니다.
`Storage Class` 를 설치하지 않고 `manual` 로 정의해도 괜찮습니다.

```bash
cd ~/k8s-1.30/k8s/utils

# 1. 설치
kubectl apply -f local-path-storage.yaml

# 2. (중요) 기본 스토리지 클래스로 지정
# 이걸 해야 Harbor가 "나 용량 줘" 할 때 자동으로 연결해줍니다.
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# 3. 확인 (local-path 옆에 (default)라고 떠야 함)
kubectl get sc
```

### 2. Harbor 설치

`./harbor-1.14.3/harbor-iamges-upload` 폴더를 설치하고자 하는 노드로 옮깁니다.

**[실행 위치: Harbor를 띄울 워커 노드 1개]**

```bash
cd harbor-iamges-upload/
sudo bash upload_images.sh
```

hostPath로 사용할 디렉토리도 생성합니다. 이때 경로는 변경해도 됩니다.

```bash
sudo mkdir -p /data/harbor
sudo chmod -R 777 /data/harbor
```

업로드가 끝나면 마스터 노드로 돌아옵니다.

**[실행 위치: K8s-Master-Node-1]**

```bash
cd harbor-1.14.3/
vi harbor_install_offline.sh
```

위 설정 정보를 해당 환경에 맞게 변경합니다.

- `EXTERNAL_HOSTNAME` : Harbor를 띄울 워커 노드 IP
- `SAVE_PATH` : 호스트 패스 실제 위치(워커 노드에 생성한 디렉토리와 동일해야 함)
- `NODE_NAME` : Harbor를 띄울 워커 노드 이름
- `STORAGE_SIZE` : Harbor 저장소 크기

설정이 끝나면 저장 후, 스크립트를 실행합니다.

```bash
sudo bash harbor_install_offline.sh
```

**해결책 (필수 적용):**

1. **Calico MTU 강제 축소:**

- `kubectl edit configmap -n kube-system calico-config`
- `veth_mtu: "0"` (자동) → **`veth_mtu: "1350"`** (수동 고정)

2.**터널링 모드 변경 (IPIP → VXLAN):**

- `kubectl edit ippool default-ipv4-ippool`
- `ipipMode: Never`, `vxlanMode: Always` 로 변경.

3.**방화벽 해제:** 워커 노드 `firewalld` 비활성화 확인.

### 2. Harbor 이미지 Pull 시, https로 가져올 때

Http를 설정했는데, Https를 호출한다면 containerd 설정을 수정해야 합니다. 모든 워커 노드에서 수정해야 합니다.

```bash
grep "config_path" /etc/containerd/config.toml

# 결과
    config_path = '/etc/containerd/certs.d:/etc/docker/certs.d'
  plugin_config_path = '/etc/nri/conf.d'
  config_path = ''
```

위와 같이 `config_path` 에 빈값이 있거나, `:` 으로 나뉘어 있다면 모두 제거합니다.

```bash
sudo vi /etc/containerd/config.toml
```

```ini
...
# 빈 값이 들어간 config_path가 있다면 제거
    config_path = '' 
...

grep 명령어를 다시 출력 시, 아래와 같이 나와야 합니다.

```bash
grep "config_path" /etc/containerd/config.toml

      config_path = '/etc/containerd/certs.d'
    plugin_config_path = '/etc/nri/conf.d'
```

그 후, tls 옵션을 끄는 설정을 추가합니다.

```bash
# 실제 하버 도메인 입력 필요
sudo mkdir -p /etc/containerd/certs.d/20.0.0.127:30002/

# 설정 추가
cat <<EOF | sudo tee /etc/containerd/certs.d/20.0.0.127:30002/hosts.toml
server = "http://20.0.0.127:30002"

[host."http://20.0.0.127:30002"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
EOF
```

서비스를 재시작합니다.

```bash
sudo systemctl restart containerd
```

-----

## 🚀 Phase 4: 접속 테스트 (PC 설정)

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
