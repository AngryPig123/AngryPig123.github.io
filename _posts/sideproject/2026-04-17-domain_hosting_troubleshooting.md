---
title: 도매인 호스팅 트러블슈팅 (Colima + k3s + NAS 환경)
description: Colima 기반 Kubernetes 환경에서 NAS의 Gitea를 Ingress로 외부 공개하는 과정에서 발생한 네트워크 격리, Traefik 백엔드 인식 문제, 라우터 포트 가로채기 이슈를 분석하고 해결한 전체 흐름을 정리한다.
date: 2026-04-17T15:00:00+09:00
updated: 2026-04-17T15:00:00+09:00
categories: [Kubernetes, DevOps, Networking, Homelab]
tags: [Kubernetes, k3s, Traefik, Ingress, Colima, Gitea, NAS, Network, PortForwarding, DevOps]
---

Gitea Ingress 트러블슈팅 전체 정리

<h2> 목적 </h2>

NAS에서 실행 중인 Gitea를 Kubernetes Ingress를 통해 외부 인터넷에 공개하는 것이 목표였다.

원하는 최종 흐름:

`Internet -> Router port 80 -> Traefik Ingress -> NAS의 Gitea`

<h2> 환경 </h2>

- Kubernetes는 macOS 위의 Colima 안에서 동작
- k3s / Traefik ingress도 Colima VM 내부에서 실행
- NAS의 Gitea 주소: `192.168.0.96:3000`
- Colima VM 주소: `192.168.5.1`
- Colima VM에서 보이는 Mac host 게이트웨이 주소: `192.168.5.2`
- 외부 도메인: `devforge.co.kr`

<h2> 처음 보인 증상 </h2>

- Kubernetes 매니페스트 구조 자체는 크게 틀리지 않았다
- Ingress, Service, 백엔드 매핑도 생성되었다
- Mac host에서는 `192.168.0.96:3000` 접속이 되었다
- 하지만 Kubernetes 클러스터 내부에서는 `192.168.0.96:3000` 접속이 실패했다
- 브라우저에서 `http://devforge.co.kr` 로 접속하면 Gitea가 아니라 라우터 관리자 페이지로 이동했다

<h2> 원인 정리 </h2>

<h3> 1. Colima 네트워크 격리 문제 </h3>

Kubernetes 클러스터가 Mac host의 LAN 위에서 직접 도는 것이 아니라 Colima VM 안에서 동작하고 있었다.

- Mac host는 `192.168.0.96:3000` 에 접근 가능

- Colima VM과 Kubernetes Pod는 `192.168.0.96:3000` 에 직접 접근 불가

  즉, Ingress 백엔드를 NAS IP로 직접 연결하는 방식은 현재 Colima 환경에서는 그대로 사용할 수 없었다.

<h3> 2. Traefik 백엔드 인식 문제 </h3>

처음에는 수동 `EndpointSlice` 를 사용해서 selector 없는 Service에 NAS 백엔드를 연결했다.

Kubernetes 자체는 서비스 엔드포인트를 보여줬지만, Traefik은 실제 라우팅 시 아래와 같은 상태가 되었다.

- `503 Service Unavailable`

- `no available server`

  이 환경에서는 수동 `EndpointSlice` 대신 전통적인 `Endpoints` 리소스로 바꾸자 Traefik이 정상적으로 백엔드를 인식했다.

<h3> 3. 라우터 관리자 페이지가 외부 요청을 가로챔 </h3>

Ingress 내부 경로를 정상화한 뒤에도 `http://devforge.co.kr` 외부 접속은 Gitea로 가지 않았다.

관찰된 증상:

- 브라우저에서 접속 시 라우터 관리자 페이지 `https://...:54443` 로 리다이렉트됨

  원인:

- 라우터의 원격 관리 / WAN 관리 기능이 켜져 있었음

- 외부에서 들어온 `80` 포트 요청을 포트포워딩하기 전에 라우터가 자기 웹관리 페이지로 처리하고 있었음

  라우터 원격 관리 기능을 끄자 이 문제가 해결되었다.

<h2> 최종 동작 구성 </h2>

Colima에서 NAS로 직접 못 가는 문제를 우회하기 위해 Mac host를 TCP 프록시로 사용했다.

최종 트래픽 흐름:

`Internet -> Router:80 -> Traefik Ingress -> Service -> Endpoints -> 192.168.5.2:13000 -> Mac TCP proxy -> 192.168.0.96:3000`



<h2> Kubernetes 리소스 구성 </h2>

<h3> Service </h3>

- 이름: `gitea-external`
- 네임스페이스: `ingress-backends`
- 포트: `80`
- TargetPort: `13000`
 
<h3> Endpoints </h3>

- 이름: `gitea-external`
- 주소: `192.168.5.2`
- 포트: `13000`

<h3> Ingress </h3>

- 이름: `gitea-ingress`
- 호스트: `devforge.co.kr`
- Ingress class: `traefik`

<h2> Mac 측 TCP 프록시 </h2>

Mac host에서 경량 TCP 프록시를 상시 실행하도록 구성했다.

- 리슨 주소: `0.0.0.0:13000`

- 포워딩 대상: `192.168.0.96:3000`

  이 프록시는 `launchd` 로 자동 실행되도록 등록했다.

  LaunchAgent:

- `~/Library/LaunchAgents/local.gitea-mac-proxy.plist`

  프록시 스크립트:

- `~/mac_tcp_proxy.py`

<h2> Gitea 애플리케이션 설정 </h2>

Gitea의 `app.ini` 에서는 외부 공개 도메인 기준으로 아래처럼 설정해야 한다.

```ini
[server]
DOMAIN = devforge.co.kr
SSH_DOMAIN = devforge.co.kr
HTTP_PORT = 3000
ROOT_URL = http://devforge.co.kr/
```

중요한 점:

- `ROOT_URL` 은 Gitea가 생성하는 링크, 리다이렉트, 외부 URL 기준을 맞추는 설정이다
- 이 값만 바꾼다고 `192.168.0.96:3000` TCP 연결이 열리는 것은 아니다
- 실제 접속 가능 여부는 네트워크 경로, 포트 바인딩, 컨테이너 포트 publish 여부에 따라 결정된다

<h2> 실제 검증한 내용 </h2>

<h3> 내부 경로 검증 </h3>

- Colima VM에서 `http://192.168.5.2:13000/` 접속 성공
- Kubernetes Pod에서 `http://192.168.5.2:13000/` 접속 성공
- `Host: devforge.co.kr` 헤더를 붙여 `http://192.168.5.1/` 로 Traefik에 요청했을 때 `200 OK` 확인

<h3> 외부 문제 확인 </h3>

`curl http://devforge.co.kr` 또는 브라우저 접속 시 예상과 다른 HTML이 나오거나 라우터 관리자 페이지로 이동하는 현상을 통해, 요청이 Gitea까지 도달하지 않는다는 점을 확인했다.

즉 이 시점의 문제는 Kubernetes 내부가 아니라 DNS 또는 라우터 처리 쪽이었다.

<h2> 최종 라우터 수정 사항 </h2>

마지막 장애 원인은 라우터였다.

해결 방법:

1. 라우터의 원격 관리 / WAN 웹관리 기능 비활성화

2. 외부 `80` 포트가 의도한 ingress 대상 쪽으로 포트포워딩되는지 확인

3. `http://devforge.co.kr` 로 재검증

   이 조치 이후 외부 접속이 정상 동작했다.

<h2> 이번 구성에서 주의할 점 </h2>

- Colima VM은 Mac host와 동일하게 홈 네트워크 장비에 접근하지 못할 수 있다
- `EndpointSlice` 와 수동 `Endpoints` 는 Traefik 동작이 같지 않을 수 있다
- `ROOT_URL` 은 애플리케이션 URL 문제를 해결하는 설정이지, TCP 연결 문제를 해결하는 설정이 아니다
- 라우터 원격 관리 기능은 포트포워딩보다 우선해서 외부 요청을 가로챌 수 있다
- 브라우저 결과와 `curl` 결과는 HTTPS 강제, 캐시, 리다이렉트 때문에 다르게 보일 수 있다

<h2> 유용한 확인 명령어 </h2>

<h3> Ingress 리소스 확인 </h3>

```bash
kubectl get svc,endpoints,ingress -n ingress-backends -o wide
kubectl describe ingress gitea-ingress -n ingress-backends
```

<h3> Mac 프록시 상태 확인 </h3>

```bash
launchctl print gui/$(id -u)/local.gitea-mac-proxy
tail -f /tmp/gitea-mac-proxy.stderr.log
```

<h3> Colima에서 확인 </h3>

```bash
colima ssh -- curl -sv http://192.168.5.2:13000/
colima ssh -- curl -sv -H 'Host: devforge.co.kr' http://192.168.5.1/
```

<h3> Pod에서 확인 </h3>

```bash
kubectl run net-debug --rm -it --restart=Never --image=curlimages/curl:8.8.0 -- curl -sv http://192.168.5.2:13000/
```

<h3> 공인 DNS / 공인 IP 확인 </h3>

```bash
dig +short devforge.co.kr
curl ifconfig.me
```

<h2> 최종 결론 </h2>

최종 문제는 Gitea 설정 파일 자체나 Kubernetes YAML 문법 문제가 아니었다.

실제 원인은 두 가지였다.

1. Colima 환경에서는 NAS IP를 Ingress 백엔드로 직접 쓰기 어려웠다

2. 라우터 관리자 웹 기능이 외부 HTTP 요청을 가로채고 있었다

   최종 해결:

- Colima 백엔드 경로는 Mac TCP 프록시와 수동 `Endpoints` 로 해결
- 외부 접속 문제는 라우터 원격 관리 비활성화와 정상 포트포워딩으로 해결