# 도메인(URL)의 실제 IP 주소 확인

사내 서버 도메인이 실제 어떤 IP를 가리키는지 확인합니다.

## 방법 A: nslookup (가장 정확함)

```bash
# DNS 서버에 등록된 정식 IP 조회
nslookup harbor.internal.company.com
```

## 방법 B: ping (간단 확인)

```bash
# 연결 확인과 동시에 IP 확인 가능
ping harbor.internal.company.com
```

## Check Point

- 사내 도메인(internal 등)은 반드시 **사내망(VPN)**에 연결된 상태여야 IP가 보입니다.
- 조회된 IP는 실제 서버가 아니라 앞단의 로드 밸런서(L4) IP일 수 있습니다 (방화벽 신청 시 이 IP 사용)
