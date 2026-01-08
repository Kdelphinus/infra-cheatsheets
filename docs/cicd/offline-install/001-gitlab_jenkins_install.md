# 1. 폐쇄망에서 GitLab 및 Jenkins 설치 및 연동

1. **가이드 환경**
   - OS: Rocky Linux 9.6
   - K8s Version: 1.30.14
   - Container Runtime: containerd (`ctr`)

2. **전제 조건**
   - Kubernetes 클러스터가 정상 동작 중이어야 합니다 (`kubectl get nodes` -> Ready).
   - Harbor가 설치되어 있어야 합니다.
   - 스토리지 클래스(`local-path`)가 구성되어 있어야 합니다.
   - 이 가이드는 마스터 노드의 `~/gitlab` , `~/jenkins` 경로에 준비되어 있다고 가정하고 시작합니다.
   - [설치 파일 위치](https://drive.google.com/drive/folders/1joMQRpZPWzKgU9BBsdxy3b0qzJMWpBC8?usp=sharing)

---

## 🚀 Phase 1: 이미지 로드 (전체 노드)

Harbor에 이미지를 업로드합니다.

**[실행 위치: Master 1]**

먼저 `upload_images_to_harbor_v2.sh` 설정 부분을 현재 환경에 맞게 변경합니다.

- `HARBOR_REGISTRY` : Harbor domain
- `HARBOR_PROJECT` : Harbor Project
- `HARBOR_USER` : ID
- `HARBOR_PASSWORD` : Password
- `USE_PLAIN_HTTP` : HTTP 접속 여부

```bash
cd ~/gitlab-18.7/images
sudo bash upload_images_to_harbor_v2.sh

cd ~/jenkins-2.528.3/images
sudo bash upload_images_to_harbor_v2.sh
```

---

## 🚀 Phase 2: 데이터 영속성 구성 (PV 설정)

`local-path` 스토리지 클래스를 사용하더라도, 프로덕션 데이터를 안전하게 보관하기 위해 **호스트 경로를 고정(HostPath)**하여
PV를 생성하는 것을 권장합니다.

**[실행 위치: Master 1]**

### 1. 호스트 디렉토리 생성 (모든 워커 노드)

데이터가 저장될 실제 폴더를 모든 워커 노드에 생성합니다.

환경에 맞춰 경로를 수정해도 됩니다.

```bash
# (각 워커 노드에서 실행하거나, Ansible 등으로 일괄 실행)
sudo mkdir -p /data/jenkins_home
sudo mkdir -p /data/gitlab_data
sudo mkdir -p /data/gitlab_pg
sudo mkdir -p /data/gitlab_redis
sudo chmod -R 777 /data

sudo mkdir -p /data/gitlab_data/minio
sudo chmod -R 777 /data/gitlab_data/minio

```

### 2. PV 생성 (Master 노드)

위에서 경로를 수정했다면, `pv-volume.yaml` 파일에서도 동일한 경로로 `sepc.hostPath.path` 값을 수정해야 합니다.

```bash
cd ~/jenkins
kubectl apply -f pv-volume.yaml
```

---

## 🚀 Phase 3: Jenkins 설치 (Master-1)

우리가 빌드한 **Custom Image (플러그인 포함)**를 사용하여 설치합니다.

**[실행 위치: K8s-Master-Node]**

### 1. 네임스페이스 생성

```bash
kubectl create namespace jenkins

```

### 2. Jenkins Helm 배포

`deploy-jenkins.sh` 파일 위에 있는 `REGISTRY_URL` 등의 변수를 환경에 맞춰 수정 후 실행합니다.

```bash
sudo bash deploy-jenkins.sh
```

### 3. Jenkins 접속 정보 확인

```bash
# 1. Pod 상태 확인 (Running이 될 때까지 대기)
watch kubectl get pods -n jenkins

# 2. 초기 관리자 계정 확인
# 초기 ID는 admin
# PW 확인
kubectl get secret -n jenkins jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode
echo ""

```

---

## 🚀 Phase 4: GitLab 설치 (Master-1)

GitLab은 리소스를 많이 사용하므로, 불필요한 기능(NGINX, Prometheus 등)을 끄고 핵심 기능만 설치합니다.
특히 **Envoy Gateway 전환**을 고려하여 Ingress 설정만 남깁니다.

**[실행 위치: K8s-Master-Node]**

### 1. 설정 파일 확인 (`install-gitlab-values.yaml`)

사용하는 환경에 맞춰 `domain` , `image` , `ingress` , `nginx-ingress` 부분을 수정해야 합니다.

```yaml
global:
  edition: ce
  hosts:
    domain: devops.internal # 내부 도메인에 맞게 변경
    https: false

  image:
    registry: 1.1.1.213:30002 # Harbor domain 맞춰 변경
    repository: library # Harbor Project 맞춰서 변경
    pullPolicy: IfNotPresent

  # [핵심] Ingress 설정
  # 1. Gateway API를 사용할 때
  ingress:
    enabled: false
    configureCertmanager: false
    tls:
      enabled: false          # TLS 비활성화 (HTTP 접속)

  # 2. 이미 설치된 ingress nginx를 사용할 때
  # ingress:
  #   enabled: true           # Ingress 객체(라우팅 규칙) 생성: YES
  #   configureCertmanager: false
  #   class: "none"           # NGINX가 채가면 안되므로 none 설정 (나중에 Envoy가 처리)
  #   tls:
  #     enabled: false          # TLS 비활성화 (HTTP 접속)

  # 3. gitlab의 ingress nginx를 사용할 떄
  # ingress:
  #   enabled: true             # Ingress 활성화
  #   configureCertmanager: false # 인증서 관리자 끔 (HTTP 사용)
  #   class: gitlab-nginx              # 내장 NGINX 사용
  #   tls:
  #     enabled: false          # TLS 비활성화 (HTTP 접속)

# [핵심] NGINX 컨트롤러 비활성화 (이미지는 받았지만 설치는 안 함)
# 1,2. Gateway API를 사용할 때 혹은 이미 설치된 ingress nginx를 사용할 때
nginx-ingress:
  enabled: false

# 3. gitlab의 ingress nginx를 사용할 때
# nginx-ingress:
#   enabled: true
#   controller:
#     ingressClassResource:
#       # [핵심] IngressClass 이름 변경 (충돌 방지)
#       name: gitlab-nginx
#       # 컨트롤러 값도 유니크하게 변경
#       controllerValue: "k8s.io/gitlab-nginx"
#     image:
#       registry: 1.1.1.213:30002 # Harbor domain
#       repository: library/ingress-nginx-controller # Harbor에 올라간 이미지
#       tag: "v1.11.8"
#       digest: ""
#     service:
#       type: NodePort
#       nodePorts:
#         http: 30080
#         https: 30443
#         ssh: 30022
...
```

### 2. GitLab Helm 배포

`install-gitlab.sh` 파일 위에 있는 설정 변수를 환경에 맞게 정의한 후 실행합니다.

```bash
./install-gitlab.sh
```

### 오류로 인한 재배포 시

오류로 인한 재배포가 필요하다면 먼저 아래 명령어로 비밀번호를 확보합니다.
혹여나 이미 비밀번호를 날렸다면, 볼륨으로 사용한 물리적 위치(가이드에선 지정한 워커 노드의 `/data/gitlab_pg` 폴더)에서
삭제 후 재생성 해야 합니다.

```bash
# 비밀번호 추출 (복사해두세요!)
kubectl get secret -n gitlab gitlab-postgresql-password -o jsonpath="{.data.postgresql-password}" | base64 -d
```

재생성 시, DB의 Password가 다시 무작위로 구성됩니다.
이때 PV에 있는 비밀번호와 맞지 새로 생성된 비밀번호가 달라 `migration job` 이 정상 동작하지 못할 때가 있습니다.
이때 아래 방법으로 비밀번호를 강제 동기화 시켜주세요.

```bash
# 1. DB 파드 이름 확인
kubectl get po -n gitlab -l app=postgresql

# 2. 파드 내부 쉘 접속 (이름이 gitlab-postgresql-0 이라고 가정)
kubectl exec -it -n gitlab gitlab-postgresql-0 -- bash
```

### 3. GitLab 접속 정보 확인

GitLab이 완전히 구동되는 데에는 약 5~10분이 소요됩니다.

```bash
# 1. 상태 모니터링
watch kubectl get pods -n gitlab

# 초기 ID는 root
# 2. 초기 root 비밀번호 확인
kubectl get secret gitlab-gitlab-initial-root-password \
  -n gitlab -ojsonpath='{.data.password}' | base64 --decode ; echo

```

---

## 🚀 Phase 5: 네트워크 및 PC 접속 설정

로드밸런서나 Gateway가 아직 구성되지 않은 상태에서 웹 접속을 확인하기 위해,
K8s 서비스의 포트를 임시로 포워딩하거나 NodePort를 확인합니다.

### 1. GitLab 서비스 노출 (임시 NodePort)

GitLab Webservice를 외부에서 접속하기 위해 NodePort로 변경합니다.

```bash
# gitlab-webservice-default 서비스 수정
kubectl patch svc gitlab-webservice-default -n gitlab -p '{"spec": {"type": "NodePort"}}'

# 할당된 포트 확인 (30000번대 포트 확인)
kubectl get svc -n gitlab gitlab-webservice-default

```

### 2. 사용자 PC Hosts 설정

사용자 PC(Windows/Mac)의 `hosts` 파일에 도메인을 등록합니다.

```text
# 예시: 워커노드 IP와 NodePort 사용 시
# <Worker-Node-IP>  gitlab.devops.internal
10.10.10.73  gitlab.devops.internal

```

이제 브라우저에서 `http://gitlab.devops.internal:<NodePort>` 로 접속하여 `root` 계정으로 로그인합니다.

---

## 🚀 Phase 6: Jenkins <-> GitLab 연동

이제 두 시스템을 연결하여 CI 파이프라인을 구성합니다.

### 1. GitLab: Access Token 발급

1. GitLab 접속 -> 우측 상단 프로필 아이콘 -> **Preferences**.
2. 좌측 메뉴 **Personal Access Tokens**.
3. **Add new token**:
   - **Name:** `jenkins-integration`
   - **Scopes:** `api` (체크)
   - **Create personal access token** 클릭 -> **토큰 값 복사**.

### 2. Jenkins: Credential 등록

1. Jenkins 접속 (`http://<NodeIP>:30000`).
2. **Manage Jenkins** -> **Credentials** -> **System** -> **Global credentials (unrestricted)**.
3. **Add Credentials**:
   - **Kind:** `GitLab API token`
   - **Scope:** `Global`
   - **API token:** (복사한 GitLab 토큰 붙여넣기)
   - **ID:** `gitlab-token-id`
   - **Description:** GitLab Connection Token
   - **Create**.

### 3. Jenkins: 시스템 설정

1. **Manage Jenkins** -> **System**.
2. 스크롤을 내려 **GitLab** 섹션 이동.
3. **Connection Name:** `cmp-gitlab` (파이프라인에서 사용할 이름).
4. **GitLab host URL:**

> **중요:** Jenkins 파드 내부에서 GitLab으로 통신해야 하므로 **K8s 내부 도메인**을 사용합니다.
> `http://gitlab-webservice-default.gitlab.svc.cluster.local:8181`

1. **Credentials:** `GitLab Connection Token` 선택.
2. **Test Connection** -> `Success` 확인 후 **Save**.

---

## 🚀 Phase 7: 파이프라인 테스트 (검증)

실제 코드가 커밋되었을 때 Jenkins가 빌드를 수행하는지 확인합니다.

### 1. Jenkins Job 생성

1. **New Item** -> 이름: `test-pipeline` -> **Pipeline** 선택.
2. **Build Triggers**: `Build when a change is pushed to GitLab` 체크.
   - Advanced -> `Secret token` 을 생성하고 복사합니다.
3. **Pipeline Script**: 아래 `image:` 경로는 실제 Harbor 경로로 변경해야 합니다.

    ```groovy
    pipeline {
          agent {
              kubernetes {
                  // yaml 병합을 통해 명시적으로 로컬 이미지를 사용하도록 지정
                  yaml """
      apiVersion: v1
      kind: Pod
      metadata:
        labels:
          app: builder
      spec:
        containers:
        # 1. 작업용 도구 (Busybox) - 로컬 이미지 사용 및 종료 방지 설정
        - name: shell
          image: '1.1.1.213:30002/library/busybox:latest'
          command: ['/bin/sh', '-c', 'sleep 86400']
          tty: true
          
        # 2. Jenkins 에이전트 (필수)
        # 스크립트 내에도 명시하여 Docker Hub 접속 시도를 원천 차단
        - name: jnlp
          image: '1.1.1.213:30002/library/inbound-agent:latest'
      """
              }
          }
          stages {
              stage('Connect Check') {
                  steps {
                      container('shell') {
                          sh 'echo "🎉 폐쇄망 Jenkins-K8s 연동 성공!"'
                          sh 'cat /etc/os-release' 
                      }
                  }
              }
          }
      }
    ```

4. **Save**

### 2. GitLab 설정 변경

1. GitLab의 우측 상단의 Profile 아이콘 -> **Admin** 클릭
2. LNB에서 **Settings** -> **Network**
3. **Outbound requests** -> 아래 항목 체크
   - `Allow requests to the local network from webhooks and integrations`
   - `Allow requests to the local network from system hooks`
4. **Save**

### 3. Webhook 등록 (GitLab)

1. Jenkins Job -> Configuration 화면의 **GitLab webhook URL**을 복사합니다.
   - 예: `http://1.1.1.213:30000/project/test-pipeline`
   - Floating IP로 되어있다면 위와 같이 project 앞부분을 내부 IP로 수정해야 합니다.

2. GitLab 프로젝트 -> **Project Settings** -> LNB의 **Webhooks** -> **Add new webhook**
3. 아래 값 설정 후, Webhook 추가
   - **URL:** 위에서 수정한 내부 주소 입력.
   - **Secret token:** Jenkins에서 생성한 Secret token.
   - **Trigger:** Push events.
   - **Add webhook**.
4. **Test** -> **Push events**
   - `HTTP 200`이 뜨면 연동 성공입니다.
   - Jenkins 대시보드에서 빌드가 수행되는지 확인하세요.

### 4. Jenkins Kubernetes Cloud 설정

Webhook을 통해 들어온 요청을 처리할 **K8s 에이전트(Pod) 연결 정보**를 설정합니다.
이 설정이 선행되어야 Jenkins가 K8s 클러스터 내부에 에이전트를 생성할 수 있습니다.

**경로:** Manage Jenkins > Clouds > New cloud

1. Kubernetes 기본 설정

   - **Name:** `Kubernetes`
   - **Kubernetes URL:** `https://kubernetes.default` (내부 API 통신용)
   - **Kubernetes Namespace:** `jenkins`
   - **Disable HTTPS certificate check:** ✅ 체크 (Enable) ->
   - **Test Connection:** `Credentials` 항목에 있는 `Test Connection` 확인
   - **Jenkins URL:** `http://jenkins.jenkins.svc.cluster.local:8080`
   - **Jenkins tunnel:** `jenkins-agent.jenkins.svc.cluster.local:50000`
   - **주의:** `kubectl get svc -n jenkins` 명령어로 50000번 포트를 가진 서비스의 정확한 이름
    (예: `jenkins` 또는 `cmp-jenkins`)을 확인하여 입력하세요.

2. Pod Templates 설정 (Global Default)

   - **Name:** `kubernetes`
   - **Namespace:** `jenkins`
   - **Labels:** `kubernetes` (파이프라인이 호출할 라벨)
   - **Usage:** `Use this node as much as possible` 선택

---

### 5. Git 설치 및 소스 코드 Push (Trigger)

모든 설정이 완료되었으므로, 실제 코드를 GitLab에 푸시하여 Jenkins 빌드를 트리거합니다.

1. Git 설치

    만약 Git이 설치되어 있지 않다면 폐쇄망용 git rpm을 사용해 git을 설치합니다.

    ```bash
    cd ~/gitlab/git-2.47.3
    tar -zxvf git_bundle_rocky96_20260107.tar.gzgit_offline_bundle
    sudo rpm -Uvh --force --nodeps git_offline_bundle/*.rpm
    ```

2. 로컬 설정

    `/etc/hosts` 파일에 GitLab 서버의 IP와 도메인을 등록합니다.

3. 프로젝트 Clone

    ```bash
    git clone http://<GitLab_도메인>/<그룹명>/<프로젝트명>.git
    # 예: git clone http://gitlab.example.com/root/test-pipeline.git
    ```

4. 테스트 파일 작성 및 Push

   - Clone 받은 폴더 안에 파일을 하나 작성합니다.
   - 작성 후 GitLab으로 푸시합니다.

   ```bash
   git add .
   git commit -m "Test Jenkins Pipeline"
   git push origin main
   ```

5. 결과 확인

   - `git push` 성공 직후 Jenkins 대시보드에서
    자동으로 빌드가 시작(`Pending` -> `Running` -> `Success`)되는지 확인합니다.
