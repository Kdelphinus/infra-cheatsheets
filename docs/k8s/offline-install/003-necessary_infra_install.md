# 3. 폐쇄망에서 Helm, Harbor, Ingress 설치

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

## 🚀 Phase 2: 필수 인프라 설치 (Ingress & Storage)

Harbor가 정상 동작하려면 **외부 접속용 문(Ingress)**과 **데이터 저장소(StorageClass)**가 필수입니다.
이것들이 없으면 Harbor 파드가 뜨다가 `Pending` 상태로 멈춥니다.

**[실행 위치: K8s-Master-Node-1]**

### 1. Local Path Provisioner (저장소)

폐쇄망에서 가장 쉬운 스토리지 해결책입니다. 로컬 디스크 경로를 PV로 씁니다.

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

## 🚀 Phase 3: Harbor 설정 파일 작성 (values.yaml)

Harbor는 드라이버에 올라가 있는 Harbor 파일을 사용합니다.

1. 이미지 폴더에서 이미지 업로드
2. 설치 폴더에 있는 설치 스크립트 실행

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
