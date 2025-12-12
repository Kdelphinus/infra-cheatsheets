# MariaDB 폐쇄망 설치용 파일 만들기

- 사용하고자 하는 환경과 동일한 서버에서 작업해야 합니다.
- 이 문서는 Rocky 9.6 버전을 기준으로 만들어졌습니다.

## 📌 1단계: 작업 환경 준비 (외부망 서버)

가져갈 파일들을 모을 디렉토리를 만듭니다.

```bash
# 작업 디렉토리 생성
mkdir -p ~/offline-dist-db/rpms
cd ~/offline-dist-db

# 외부망 서버 자체를 위한 도구 설치 (이미지 다운로드용)
sudo dnf install -y yum-utils
```

-----

## 📌 2단계: 레포지토리 등록 (소스 확보)

Rocky 9.6 기본 레포지토리 외에 우리가 필요한 **MariaDB** 공식 저장소를 등록합니다.

```bash
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
dnf download --resolve --alldeps --arch=x86_64 --destdir=./rpms \
    vim net-tools telnet wget curl git jq tar unzip \
    conntrack-tools socat ipvsadm ipset \
    keepalived haproxy \
    chrony policycoreutils-python-utils \
    createrepo_c modulemd-tools
```

```bash
# 2. MariaDB 10.11 & Galera Cluster
# 위에서 10.11 repo를 등록했으므로, 이 명령어가 10.11.14 버전을 받아옵니다.
dnf download --resolve --alldeps --arch=x86_64 --destdir=./rpms \
    MariaDB-server-10.11.14 \
    MariaDB-client-10.11.14 \
    MariaDB-common-10.11.14 \
    MariaDB-shared-10.11.14 \
    MariaDB-backup-10.11.14 \
    galera-4 rsync
```

-----

## 📌 4단계: 최종 포장 (압축)

이제 모든 파일이 `~/offline-dist`에 모였습니다. 이걸 하나로 묶어서 USB나 망연계 솔루션을 통해 넘기시면 됩니다.

```bash
cd ~
tar -zcvf offline-dist-rocky9.6-k8s1.30.tar.gz offline-dist/
```
