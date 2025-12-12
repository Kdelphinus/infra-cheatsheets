# 1. Kubernetes & MariaDB 폐쇄망 설치용 파일 만들기

- 이 문서는 Rocky 9.6 버전을 기준으로 만들어졌습니다.
- 사용하고자 하는 환경과 동일한 서버에서 작업해야 합니다.
- K8s, MariaDB 외에도 Harbor 등의 설치 파일도 생성하니 확인하고 필요한 서비스만 받아야 합니다.

## 📌 1단계: 작업 환경 준비 (외부망 서버)

가져갈 파일들을 모을 디렉토리를 만듭니다.

```bash
# 작업 디렉토리 생성
mkdir -p ~/offline-dist-split
BASE_DIR=~/offline-dist-split
mkdir -p $BASE_DIR/{common,db,k8s}
mkdir -p $BASE_DIR/k8s/{rpms,images,charts,binaries,utils}
mkdir -p $BASE_DIR/db/rpms
mkdir -p $BASE_DIR/common/rpms

# 외부망 서버 자체를 위한 도구 설치 (이미지 다운로드용)
sudo dnf install -y yum-utils device-mapper-persistent-data lvm2
```

```bash
# 1. Docker 설치 (인터넷에서 바로 설치)
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 2. Docker 실행 및 자동 시작 설정
sudo systemctl enable --now docker

# 3. 권한 설정 (sudo 없이 docker 명령어 쓰기 위함)
sudo usermod -aG docker $USER

# 4. 그룹 권한 즉시 적용 (로그아웃 안 해도 됨)
newgrp docker

# 5. 확인 (잘 되면 버전 정보가 뜹니다)
docker version
```

-----

## 📌 2단계: 레포지토리 등록 (소스 확보)

Rocky 9.6 기본 레포지토리 외에 우리가 필요한 **Docker, K8s, MariaDB** 공식 저장소를 등록합니다.

```bash
# Docker Repo (K8s 노드에 깔 'containerd.io' 확보를 위해 필수!)
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Kubernetes v1.30 Repo
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF

# MariaDB 10.11 Repo (시리즈 전체 지정)
cat <<EOF | sudo tee /etc/yum.repos.d/mariadb.repo
[mariadb]
name = MariaDB
baseurl = https://rpm.mariadb.org/10.11/rhel/\$releasever/\$basearch
gpgkey=https://rpm.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
EOF

# 레포지토리 정보 갱신
sudo dnf makecache
```

-----

## 📌 3단계: RPM 패키지 다운로드 (핵심)

`--resolve --alldeps` 옵션으로 의존성 라이브러리까지 싹 긁어옵니다. **여기가 제일 중요합니다.**

```bash
# 1. 시스템 기본 도구 및 HA(고가용성) 도구
# createrepo_c: 폐쇄망 내부에서 repo 만들 때 필수! (절대 누락 금지)
# haproxy, keepalived: Master/DB 3중화 필수
dnf download --resolve --alldeps --arch=x86_64 --destdir=./common/rpms \
    vim net-tools telnet wget curl git jq tar unzip \
    conntrack-tools socat ipvsadm ipset \
    keepalived haproxy \
    chrony \
    createrepo_c
```

`--arch` 옵션에 설치 환경에 맞는 CPU 타입을 입력해야 합니다.

```bash
# 2. Container Runtime (K8s용)
# Docker 전체가 아니라 'containerd.io'가 핵심입니다.
# 다만, 나중에 디버깅용으로 CLI 도구가 있으면 편하므로 docker-ce-cli도 같이 받습니다.
dnf download --resolve --alldeps --arch=x86_64 --destdir=./k8s/rpms \
    containerd.io docker-ce-cli
```

```bash
# 3. Kubernetes (v1.30.x)
dnf download --resolve --alldeps --arch=x86_64 --destdir=./k8s/rpms \
    kubelet-1.30.* kubeadm-1.30.* kubectl-1.30.* \
    containerd.io docker-ce-cli
```

```bash
# 4. MariaDB 10.11 & Galera Cluster
# 위에서 10.11 repo를 등록했으므로, 이 명령어가 10.11.14 버전을 받아옵니다.
dnf download --resolve --alldeps --arch=x86_64 --destdir=./db/rpms \
    MariaDB-server-10.11.14 \
    MariaDB-client-10.11.14 \
    MariaDB-common-10.11.14 \
    MariaDB-shared-10.11.14 \
    MariaDB-backup-10.11.14 \
    galera-4 rsync
```

-----

## 📌 4단계: 바이너리 파일 다운로드

RPM으로 설치 안 되는 실행 파일들입니다.

```bash
# Helm v3 (K8s에 Harbor 배포 시 필수)
wget https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz -P ./k8s/binaries/

# cri-dockerd (containerd만 쓴다면 필요 없지만, 비상용으로 받아둠)
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.10/cri-dockerd-0.3.10.amd64.tgz -P ./k8s/binaries/

# Calico YAML 다운로드
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Ingress nginx 다운로드
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml -O ingress-nginx.yaml

# Local path storage 다운로드
wget https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml -O local-path-storage.yaml

# Yaml file 이동
mv calico.yaml ingress-nginx.yaml local-path-storage.yaml ./k8s/utils/
```

Helm을 설치합니다.

```bash
# 1. 방금 받은 Helm 압축 파일 풀기
tar -zxvf ./k8s/binaries/helm-v3.14.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
rm -rf linux-amd64

# 2. 잘 설치됐는지 확인
helm version

# 3. Harbor Helm Chart 다운로드
# Helm 레포 추가
helm repo add harbor https://helm.goharbor.io
helm repo update
helm pull harbor/harbor --destination ./k8s/charts/
```

-----

## 📌 5단계: 컨테이너 이미지 다운로드 (스크립트 실행)

K8s와 네트워크 플러그인(Calico) 이미지를 받아 `.tar`로 저장합니다. 아래 명령어를 한 번에 실행시켜야 합니다.

```bash
# 이미지 리스트 정의
K8S_IMAGES=(
    "registry.k8s.io/kube-apiserver:v1.30.0"
    "registry.k8s.io/kube-controller-manager:v1.30.0"
    "registry.k8s.io/kube-scheduler:v1.30.0"
    "registry.k8s.io/kube-proxy:v1.30.0"
    "registry.k8s.io/pause:3.9"
    "registry.k8s.io/etcd:3.5.15-0"
    "registry.k8s.io/coredns/coredns:v1.11.3"
)
CALICO_IMAGES=(
    "docker.io/calico/cni:v3.27.0"
    "docker.io/calico/node:v3.27.0"
    "docker.io/calico/kube-controllers:v3.27.0"
)
HARBOR_IMAGES=(
    "goharbor/harbor-core:v2.10.0"
    "goharbor/harbor-db:v2.10.0"
    "goharbor/harbor-jobservice:v2.10.0"
    "goharbor/harbor-portal:v2.10.0"
    "goharbor/harbor-registryctl:v2.10.0"
    "goharbor/registry-photon:v2.10.0"
    "goharbor/harbor-exporter:v2.10.0"
    "goharbor/redis-photon:v2.10.0"
    "goharbor/trivy-adapter-photon:v2.10.0"
)
ADDON_IMAGES=(
    "registry.k8s.io/ingress-nginx/controller:v1.10.0"
    "registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.4.0"
    "rancher/local-path-provisioner:v0.0.26"
    "busybox:latest"
)

ALL_IMAGES=("${K8S_IMAGES[@]}" "${CALICO_IMAGES[@]}" "${HARBOR_IMAGES[@]}" "${ADDON_IMAGES[@]}")

# 저장 루프
for img in "${ALL_IMAGES[@]}"; do
    echo "Processing $img ..."
    docker pull $img
    filename=$(echo $img | sed 's/\//_/g' | sed 's/:/_/g').tar
    docker save $img -o ./k8s/images/$filename
done
```

-----

## 📌 6단계: 최종 포장 (압축)

이제 모든 파일이 `~/offline-dist`에 모였습니다. 이걸 하나로 묶어서 USB나 망연계 솔루션을 통해 넘기시면 됩니다.

```bash
cd ~

# 1. DB 노드용 번들 (Common + DB) -> 용량 매우 작음 (수백 MB)
# DB 노드에는 이것만 가져가면 됩니다.
tar -zcvf dist-for-db-nodes.tar.gz offline-dist-split/common offline-dist-split/db

# 2. K8s 노드용 번들 (Common + K8s + Images) -> 용량 큼 (수 GB)
# K8s Master/Worker 노드에는 이것만 가져가면 됩니다.
tar -zcvf dist-for-k8s-nodes.tar.gz offline-dist-split/common offline-dist-split/k8s
```
