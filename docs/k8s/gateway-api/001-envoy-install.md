# 🚀 Envoy 설치

2026년 3월부터 `Ingress Nginx` 에 대한 공식 지원이 종료됩니다.
이에 따라 Kubernetes의 `Gateway API` 와 `Envoy` 를 사용하여 합니다.

이 문서는 폐쇄망을 기준으로 합니다.
[설치 파일](https://drive.google.com/drive/folders/1joMQRpZPWzKgU9BBsdxy3b0qzJMWpBC8?hl=ko)
의 `envoy-1.36.3` 폴더를 받아주세요.

## 0. 아키텍처 개요 (Standard Architecture)

쿠버네티스 보안 및 네트워크 표준을 준수하는 구성입니다.

- **Network:** `hostNetwork: false` (Pod는 K8s 내부망 사용, 노드 네트워크와 격리)
- **Service:** `type: LoadBalancer` (외부 트래픽 진입점)
- **Traffic Flow:**
`Client` -> `External IP (LB)` -> `Service (80/443)` -> `Envoy Pod (10080/10443)`
-> `Backend Pod`

## 1. 설치

**[실행 위치: Master 1, Worker 1~3 전체]**

전체 노드에 `envoy` 이미지들을 로드합니다.

```bash
cd ./envoy-1.36.3
sudo bash ./images/upload_images.sh
```

**[실행 위치: Master 1]**

`install_envoy-gateway.sh` 스크립트에서 아래 변수를 환경에 맞게 변경합니다.
(기본값을 사용해도 괜찮습니다.)

- `NAMESPACE:` Gateway Namespace
- `CONTROLLER_CHART:` Controller Chart(기본값 고정)
- `INFRA_CHART:` Infra Chart(기본값 고정)
- `GW_NAME:` Gateway Name
- `IMG_GATEWAY:` Gateway 이미지(기본값 고정)
- `IMG_PROXY:` Proxy 이미지(기본값 고정)
- `GW_CLASS_NAME:` 클러스터 레벨 리소스 이름

이때 `NAMESPACE` 나 `GW_NAME` 를 변경했다면, `HTTPRoute` 파일에도 변경해야 합니다.

수정이 끝나면 스크립트를 실행합니다.

```bash
sudo bash install_envoy-gateway.sh
```

## 2. 배포 후 상태 확인 및 IP 할당

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

## 🛠️ [Case C] 수동 IP 할당 명령어

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

## 3. 라우팅(HTTPRoute) 설정 및 검증

Gateway가 정상적으로 떴다면, 애플리케이션 연결 규칙(`HTTPRoute`)을 점검합니다.
이 검증은 서비스에 접근하지 못할 때 진행해도 됩니다.

> 서비스보다 먼저 envoy를 설치하기 때문에 현재는 생성된 HTTPRoute 자원이 없습니다.

## ✅ 체크리스트 1: Gateway 이름 일치 여부

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

## ✅ 체크리스트 2: 백엔드 포트 (Connection Refused)

Envoy는 서비스(ClusterIP) 포트가 아닌 **파드(Pod)의 실제 컨테이너 포트**로 접속을 시도합니다.

- **증상:** 503 Service Unavailable 또는 Connection Refused
- **해결:** `HTTPRoute`의 `backendRefs` 포트를
**TargetPort(실제 앱 포트, 예: 8080)** 로 설정해야 합니다.

```bash
# 포트를 80 -> 8080으로 변경하는 예시
kubectl patch httproute <ROUTE_NAME> -n <NAMESPACE> --type='json' \
  -p='[{"op": "replace", "path": "/spec/rules/0/backendRefs/0/port", "value": 8080}]'
```

## ✅ 체크리스트 3: 경로 재작성 (URL Rewrite)

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

## 4. 운영 및 로그 확인

Envoy Gateway는 동적으로 리소스를 관리하므로 파드 이름이 변경됩니다.
**Label Selector(`-l`)**를 사용하여 로그를 확인하는 것이 표준입니다.

## 📋 프록시(Data Plane) 로그

실제 트래픽 처리, 접속 오류 확인 시 사용합니다.

```bash
# Envoy Proxy 로그 실시간 확인
kubectl logs -n envoy-gateway-system -f -l gateway.envoyproxy.io/owning-gateway-name=cmp-gateway

```

## 🧠 컨트롤러(Control Plane) 로그

Gateway 설정 변환, 배포 실패 원인 분석 시 사용합니다.

```bash
# Gateway Controller 로그 확인
kubectl logs -n envoy-gateway-system -f -l app.kubernetes.io/name=envoy-gateway

```
