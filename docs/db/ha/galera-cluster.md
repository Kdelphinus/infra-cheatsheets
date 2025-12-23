# 📘 [폐쇄망] MariaDB 10.11 Galera Cluster 설치 가이드

**환경:** Rocky Linux 9.6 / MariaDB 10.11 (Offline)
**디렉토리:** `mariadb-10.11.14-for-air-gapped` 구조 기반
**계정:** 일반 사용자 (sudo 권한 필수)

## 1\. 구성 정보 (Topology)

* **Cluster Name:** `my_galera_svc`
* **DB Root User:** `root`

| 호스트명 (Hostname) | 역할 | IP (작성 필요) | 비고 |
| :--- | :--- | :--- | :--- |
| **galera-cluster-1** | **Primary** (Bootstrap) | `IP_1` | 최초 클러스터 시작 |
| **galera-cluster-2** | Member | `IP_2` | |
| **galera-cluster-3** | Member | `IP_3` | |

-----

## 2\. OS 및 네트워크 설정 (3대 공통)

### 2.1 호스트 파일 등록

IP 확정 후 3대 서버 모두 동일하게 수정합니다.

```bash
# sudo vi /etc/hosts
# [IP주소]          [호스트명]
192.168.XXX.XX   galera-cluster-1
192.168.XXX.XX   galera-cluster-2
192.168.XXX.XX   galera-cluster-3
```

### 2.2 호스트네임 및 방화벽 설정

```bash
# 1. 호스트네임 변경 (각 서버 번호에 맞게 실행)
sudo hostnamectl set-hostname galera-cluster-1

# 2. SELinux Permissive 변경 (필수)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# 3. 방화벽 포트 오픈
sudo firewall-cmd --permanent --add-port={3306,4567,4568,4444}/tcp
sudo firewall-cmd --permanent --add-port=4567/udp
sudo firewall-cmd --reload
```

-----

## 3\. 설치 (Offline) - 3대 공통

`mariadb-10.11.14-for-air-gapped` 폴더가 각 서버의 홈 디렉토리(예: `/home/rocky`)에 업로드되어 있어야 합니다.

### 3.1 Rocky Linux 기본 MariaDB 모듈 비활성화 (매우 중요)

Rocky 9에 내장된 구버전 MariaDB와 충돌하지 않도록 모듈을 끕니다.

```bash
sudo dnf module disable mariadb -y --disablerepo=*
```

만약 아래와 같은 오류가 났다면 충돌 위험이 없으므로 넘어가면 됩니다.

```bash
Unable to resolve argument mariadb
Error: Problems in request:
missing groups or modules: mariadb
```

### 3.2 공통 의존성 패키지 설치 (`common`)

먼저 베이스가 되는 라이브러리들을 설치합니다.

```bash
# 1. common 폴더로 이동
cd ~/mariadb-10.11.14-for-air-gapped/common/rpms

# 2. 일괄 설치
# --disablerepo=* : 인터넷 연결 시도 차단
# --skip-broken : 중복/충돌 패키지가 있을 경우 무시하고 진행
sudo dnf install -y ./*.rpm --disablerepo=* --skip-broken
```

### 3.3 DB 및 Galera 패키지 설치 (`db`)

실제 DB 서버와 Galera, 백업 도구를 설치합니다.

```bash
# 1. db 폴더로 이동
cd ~/mariadb-10.11.14-for-air-gapped/db/rpms

# 2. 일괄 설치
sudo dnf install -y ./*.rpm --disablerepo=* --skip-broken
```

### 3.4 서비스 등록

설치가 완료되면 서비스를 등록합니다. (아직 시작하지 마세요\!)

```bash
sudo systemctl enable mariadb
```

저장 경로를 변경하려면 미리 경로에 디렉토리를 생성해야 합니다.

```bash
# 1. 데이터 저장소 폴더 생성 (예: /app/mariadb_data)
sudo mkdir -p /app/mariadb_data

# 2. 소유권 변경 (mysql 계정은 RPM 설치 시 자동 생성됨)
sudo chown -R mysql:mysql /app/mariadb_data
sudo chmod 750 /app/mariadb_data

# 3. (중요) DB 초기화
# 경로를 바꿨기 때문에 기본 생성된 데이터가 없습니다. 수동으로 시스템 테이블을 깔아줘야 합니다.
sudo mysql_install_db --user=mysql --datadir=/app/mariadb_data
```

-----

## 4\. Galera 설정 파일 작성 (3대 공통)

`/etc/my.cnf.d/01-galera.cnf` 파일을 생성합니다.

```bash
sudo vi /etc/my.cnf.d/01-galera.cnf
```

**[아래 내용 붙여넣기 - ⚠️ 서버별 IP 수정 필수\!]**

```ini
[mariadb]
# ----------------------------------------------
# 1. 기본 및 호환성 설정 (Basic & Compatibility)
# ----------------------------------------------
datadir=/app/mariadb_data
# 소켓 파일은 가급적 기본 위치 유지 권장 (클라이언트 접속 편의성)
# 만약 소켓도 옮기고 싶다면 socket=/app/mariadb_data/mysql.sock 추가

bind-address=0.0.0.0
default_storage_engine=InnoDB
binlog_format=ROW
innodb_autoinc_lock_mode=2

# [튜닝] 대소문자 구분 안 함 (1 = 무시, 소문자로 저장)
# 주의: 이 설정은 DB 초기화 전에 적용되어야 합니다.
lower_case_table_names=1

# [튜닝] 커넥션 증설 (기본 151 -> 1000)
max_connections=1000

# [튜닝] SQL Mode 완화 (ONLY_FULL_GROUP_BY 제거)
# 쿼리 작성 시 GROUP BY 절 제약을 완화하여 호환성 확보
sql_mode="STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION"

# ----------------------------------------------
# 2. Galera Cluster 설정 (Provider)
# ----------------------------------------------
wsrep_on=ON
# 설치된 라이브러리 경로 (경로 확인 필수)
wsrep_provider=/usr/lib64/galera-4/libgalera_smm.so

# ----------------------------------------------
# 3. 클러스터 공통 설정 (3대 서버 모두 동일)
# ----------------------------------------------
wsrep_cluster_name="my_galera_svc"
# 3대 노드의 IP를 공백 없이 콤마로 나열
wsrep_cluster_address="gcomm://IP_1,IP_2,IP_3"

# ----------------------------------------------
# 4. 노드별 고유 설정 (⚠️ 서버마다 수정 필수!)
# ----------------------------------------------
# 현재 서버의 IP 주소
wsrep_node_address="본인_서버_IP"
# 현재 서버의 호스트명 (galera-cluster-1, 2, 3)
wsrep_node_name="galera-cluster-X"

# ----------------------------------------------
# 5. 동기화 방식 (SST)
# ----------------------------------------------
wsrep_sst_method=mariabackup
```

-----

## 5\. 클러스터 기동 (순서 준수\!)

### 🚀 [Step 1] galera-cluster-1 (Bootstrap)

**반드시 1번 서버에서 가장 먼저 실행합니다.**

```bash
# 1. 클러스터 초기화 (New Cluster)
sudo galera_new_cluster

# 2. 클러스터 상태 확인 (Size가 1이어야 함)
sudo mariadb -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

### 🚀 [Step 2] galera-cluster-2

```bash
# 1. 서비스 시작 (자동으로 1번에 합류)
sudo systemctl start mariadb

# 2. 확인 (Size가 2로 늘어나야 함)
sudo mariadb -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

### 🚀 [Step 3] galera-cluster-3

```bash
# 1. 서비스 시작
sudo systemctl start mariadb

# 2. 최종 확인 (Size가 3이어야 함)
sudo mariadb -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

-----

## 6\. 검증 (Validation)

1. **Node 1에서 DB 생성:**

    ```bash
    sudo mariadb -u root -e "CREATE DATABASE galera_test_db;"
    ```

2. **Node 3에서 확인:**

    ```bash
    sudo mariadb -u root -e "SHOW DATABASES;"
    ```

      * `galera_test_db`가 보이면 3중화 성공입니다.

3. **K8s 노드와 연결하기**

만약, 다른 노드에 있는 K8s가 접속해야 한다면 IP 허용 규칙을 추가해야 합니다.

```sql
-- K8s 파드들이 사용할 계정 생성
-- '20.%'는 20.으로 시작하는 모든 IP 허용
CREATE USER 'k8s_app_user'@'20.%' IDENTIFIED BY 'K8s_Passw0rd!';
GRANT ALL PRIVILEGES ON *.* TO 'k8s_app_user'@'20.%';
FLUSH PRIVILEGES;
```

테스트 혹은 개발 환경이라면 편의를 위해 `root` 계정을 열어도 괜찮지만 비권장 사항입니다.

```sql
-- 1. root 계정을 20.x.x.x 대역에서 접속 허용
-- IDENTIFIED BY 뒤에 '비밀번호'를 꼭 지정해야 합니다.
GRANT ALL PRIVILEGES ON *.* TO 'root'@'20.%' IDENTIFIED BY '설정할_비밀번호' WITH GRANT OPTION;

-- 2. 적용
FLUSH PRIVILEGES;
```

적용되었는지는 임시 파드를 만들어서 확인합니다.

```bash
kubectl run tmp-shell --rm -it \
  --image=docker.io/library/busybox:latest \
  --image-pull-policy=Never \
  --restart=Never -- sh
```

```bash
telnet <IP_1> 3306

# 아래처럼 Connected와 깨진 문자열이 나오면 연결된 것입니다.
Connected to 20.0.0.140
Z
5.5.5-10.11.14-MariaDBI5j;$CcK�+4cuF:1'nv)cmysql_native_password
```
