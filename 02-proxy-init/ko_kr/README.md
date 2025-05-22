# P프록시-Init

`linkerd-init` 컨테이너는 메시(mesh)에 포함된 각 파드에 Kubernetes Init 컨테이너로 추가되며, 애플리케이션 컨테이너보다 먼저 실행됩니다. 이 컨테이너는 `iptables` 규칙을 설정해 파드로 들어오고 나가는 모든 TCP 트래픽을 Linkerd 프록시로 우회합니다.

## 참고 자료
- https://linkerd.io/2-edge/reference/architecture/
- https://github.com/linkerd/linkerd2-proxy-init/blob/main/proxy-init/cmd/root.go

# 사전 준비 사항

- Unix-스타일 셸이 가능한 macOS/Linux/Windows
- 로컬 Kubernetes 클러스터용 k3d (v5+)
- kubectl (v1.25+)
- Helm (v3+)
- 인증서 생성을 위한 Smallstep (step) CLI

# 튜토리얼

## 1. 로컬 Kubernetes 클러스터 생성

k3d와 `cluster.yaml` 파일을 사용해 경량 Kubernetes 클러스터를 띄웁니다.

```
k3d cluster create --kubeconfig-update-default \
  -c ./cluster.yaml
```

## 2. ID 인증서(Trust Anchor & Issuer) 생성

Linkerd는 mTLS ID를 위해 신뢰 루트(루트 CA)와 발급자(중간 CA)가 필요합니다.

```
step certificate create root.linkerd.cluster.local ./certificates/ca.crt ./certificates/ca.key \
    --profile root-ca \
    --no-password \
    --insecure
step certificate create identity.linkerd.cluster.local ./certificates/issuer.crt ./certificates/issuer.key \
    --profile intermediate-ca \
    --not-after 8760h \
    --no-password \
    --insecure \
    --ca ./certificates/ca.crt \
    --ca-key ./certificates/ca.key
```

## 3. Helm으로 Linkerd 설치

```
helm repo add linkerd-edge https://helm.linkerd.io/edge
helm repo update
helm install linkerd-crds linkerd-edge/linkerd-crds \
  -n linkerd --create-namespace --set installGatewayAPI=true
helm upgrade --install linkerd-control-plane \
  -n linkerd \
  --set-file identityTrustAnchorsPEM=./certificates/ca.crt \
  --set-file identity.issuer.tls.crtPEM=./certificates/issuer.crt \
  --set-file identity.issuer.tls.keyPEM=./certificates/issuer.key \
  --set controllerLogLevel=debug \
  --set policyController.logLevel=debug \
  --set policyController.logLevel=debug \
  linkerd-edge/linkerd-control-plane
```

## 4. Linkerd destination 파드 확인

`linkerd-destination` 파드를 살펴보면 `linkerd-init` 컨테이너와 Helm 차트 기본값으로 전달된 인자들을 볼 수 있습니다.

```
kubectl describe pod -n linkerd                  linkerd-destination-8696d67545-4d4hj 
Name:             linkerd-destination-8696d67545-4d4hj
Namespace:        linkerd
...
Init Containers:
  linkerd-init:
    Container ID:    containerd://30f1e3964e09df03c043c38911fa521766cc71b0061ff12a8db53730ea14f4ec
    Image:           cr.l5d.io/linkerd/proxy-init:v2.4.2
    Image ID:        cr.l5d.io/linkerd/proxy-init@sha256:fa4ffce8c934f3a6ec89e97bda12d94b1eb485558681b9614c9085e37a1b4014
    Port:            <none>
    Host Port:       <none>
    SeccompProfile:  RuntimeDefault
    Args:
      --ipv6=false
      --incoming-proxy-port
      4143
      --outgoing-proxy-port
      4140
      --proxy-uid
      2102
      --inbound-ports-to-ignore
      4190,4191,4567,4568
      --outbound-ports-to-ignore
      443,443
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Sun, 18 May 2025 23:42:27 +0900
      Finished:     Sun, 18 May 2025 23:42:27 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /run from linkerd-proxy-init-xtables-lock (rw)
    ...
```

## 5. 디버그 컨테이너 배포

파드를 재시작하지 않고 iptables 규칙을 확인하려면 netadmin 프로파일이 적용된 Ubuntu 디버그 컨테이너를 띄웁니다.

```
kubectl debug -n linkerd deploy/linkerd-destination \
  -it \
  --image=ubuntu:22.04 \
  --target=destination \
  --profile=netadmin \
  -- bash -il
```

컨테이너 안에서 `iptables`를 설치합니다.

```
apt-get update && apt-get install -y iptables
```

# 6. iptables 확인

이제 디버그 컨테이너에서 iptables 체인을 볼 수 있습니다. 먼저 인바운드 `PREROUTING` 체인을 확인해 봅시다.

```
iptables-legacy -t nat -L PREROUTING -n -v
Chain PREROUTING (policy ACCEPT 3095 packets, 186K bytes)
 pkts bytes target               prot opt in     out     source               destination         
12412  745K PROXY_INIT_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* proxy-init/install-proxy-init-prerouting */
```

모든 수신 패킷은 먼저 `PROXY_INIT_REDIRECT` 체인으로 전달됩니다.

```
iptables-legacy -t nat -L PROXY_INIT_REDIRECT -n -v
Chain PROXY_INIT_REDIRECT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
 3096  186K RETURN     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            multiport dports 4190,4191,4567,4568 /* proxy-init/ignore-port-4190,4191,4567,4568 */
 9320  559K REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* proxy-init/redirect-all-incoming-to-proxy-port */ redir ports 4143
```

- 첫 번째 규칙은 포트 `4190`, `4191`, `4567`, 4568로 향하는 트래픽을 우회시킵니다.
- 두 번째 규칙은 그 외 모든 인바운드 TCP 트래픽을 프록시의 인바운드 리스너(포트 `4143`)로 리다이렉트합니다.

```
iptables-legacy -t nat -L PROXY_INIT_REDIRECT -n -v
Chain PROXY_INIT_REDIRECT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
 3096  186K RETURN     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            multiport dports 4190,4191,4567,4568 /* proxy-init/ignore-port-4190,4191,4567,4568 */
 9320  559K REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* proxy-init/redirect-all-incoming-to-proxy-port */ redir ports 4143
```

아웃바운드 트래픽은 `OUTPUT` 체인을 거칩니다.

```
iptables-legacy -t nat -L OUTPUT     -n -v
Chain OUTPUT (policy ACCEPT 9360 packets, 562K bytes)
 pkts bytes target             prot opt in     out     source               destination         
 9364  563K PROXY_INIT_OUTPUT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* proxy-init/install-proxy-init-output */
```

- Similarly to the inbound, all outbound traffic will hits the `PROXY_INIT_OUTPUT` chain. 

```
iptables-legacy -t nat -L PROXY_INIT_OUTPUT   -n -v
Chain PROXY_INIT_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
 9320  559K RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 2102 /* proxy-init/ignore-proxy-user-id */
    0     0 RETURN     all  --  *      lo      0.0.0.0/0            0.0.0.0/0            /* proxy-init/ignore-loopback */
   26  1560 RETURN     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            multiport dports 443,6443 /* proxy-init/ignore-port-443,6443 */
    4   240 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* proxy-init/redirect-all-outgoing-to-proxy-port */ redir ports 4140
```

- UID 2102(프록시 프로세스)가 생성한 트래픽은 리다이렉트에서 제외됩니다.
- 루프백 트래픽을 제외합니다.
- 포트 `443`(HTTPS)와 `6443`(Kubernetes API 서버)으로 향하는 트래픽은 제외됩니다.
- 그 외 모든 아웃바운드 TCP 트래픽을 프록시의 아웃바운드 리스너(포트 `4140`)로 리다이렉트합니다.