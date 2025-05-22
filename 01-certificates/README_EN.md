# 인증서

Linkerd는 메시에 참여한 파드 간 모든 TCP 트래픽에 대해 mTLS를 자동으로 활성화합니다. 이를 위해 컨트롤 플레인이 올바르게 동작하려면 몇 가지 인증서가 필요합니다. 인증서는 설치 시 직접 제공하거나, Cert-Manager·Trust-Manager 같은 서드파티 도구로 관리할 수 있습니다.

## 참고 링크
- https://linkerd.io/2-edge/tasks/generate-certificates/
- https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-review-v1/
- https://github.com/linkerd/linkerd2-proxy/blob/main/linkerd/proxy/identity-client/src/certify.rs
- https://github.com/linkerd/linkerd2-proxy/blob/main/linkerd/proxy/spire-client/src/lib.rs
- https://github.com/linkerd/linkerd2-proxy/blob/main/linkerd/app/src/identity.rs
- https://github.com/linkerd/linkerd2/blob/main/controller/identity/validator.go
- https://github.com/linkerd/linkerd2/blob/main/proxy-identity/main.go

# 필요 조건

- Unix-스타일 셸이 가능한 macOS/Linux/Windows
- 로컬 Kubernetes 클러스터용 k3d (v5+)
- kubectl (v1.25+)

# 튜토리얼

## 1. 로컬 Kubernetes 클러스터 생성

`cluster.yaml`을 사용해 k3d로 경량 Kubernetes 클러스터를 띄웁니다.

```
k3d cluster create --kubeconfig-update-default \
  -c ./cluster.yaml
```

## 2. 아이덴티티(Identity) 인증서 생성

Linkerd mTLS에는 신뢰 앵커(루트 CA) 와 발급자(중간 CA) 가 필요합니다.

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

## 3. 루트 신뢰 앵커(Root Trust Anchor) 인증서

Linkerd의 루트 신뢰 앵커는 서비스 메시 내 모든 인증서의 최종 신뢰 지점을 제공하는 공개 CA 인증서입니다. 이 인증서는 직접 워크로드 인증서를 발급하지 않고, 대신 중간 CA 를 서명하여 워크로드 인증서를 발급하도록 위임합니다. 이렇게 하면 여러 클러스터가 각자 발급자를 운영하면서도 동일한 루트 앵커를 통해 메시 전반의 신뢰를 유지할 수 있으며, 루트 키를 일상 업무에 노출하지 않아도 됩니다.

루트 앵커 인증서(공개키만 포함)는 `linkerd-identity-trust-roots`라는 ConfigMap 에 저장됩니다. 비밀 키가 없으므로 공개 저장이 가능하며, 모든 중간·엔티티 인증서의 신뢰 부트스트랩에 사용됩니다. 기업 환경에서는 자체 PKI로 중간 CA를 생성하여 이 루트에 연결하는 경우가 많습니다.

새 Linkerd 프록시가 워크로드 파드에 주입(inject)되면, 환경 변수와 볼륨을 통해 신뢰 구성을 전달받습니다.

```
linkerd-proxy:
    Container ID:    containerd://f348b4bebec14d557c44951f309e07fac969de2ea93f20e9d1920b4a8e02180e
    Image:           cr.l5d.io/linkerd/proxy:edge-25.5.3
    ...
    Environment:
     ...
      LINKERD2_PROXY_IDENTITY_DIR:                               /var/run/linkerd/identity/end-entity
      LINKERD2_PROXY_IDENTITY_TRUST_ANCHORS:                     <set to the key 'ca-bundle.crt' of config map 'linkerd-identity-trust-roots'>  Optional: false
      LINKERD2_PROXY_IDENTITY_TOKEN_FILE:                        /var/run/secrets/tokens/linkerd-identity-token
      ...
    Mounts:
      /var/run/linkerd/identity/end-entity from linkerd-identity-end-entity (rw)
      /var/run/secrets/tokens from linkerd-identity-token (rw)
...
Volumes:
  trust-roots:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      linkerd-identity-trust-roots
    Optional:  false
  linkerd-identity-token:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  86400
  linkerd-identity-end-entity:
    Type:        EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:      Memory
    SizeLimit:   <unset>
```

프록시가 시작되면 `LINKERD2_PROXY_IDENTITY_TRUST_ANCHORS`로 지정된 앵커를 로드합니다. 다음으로 `LINKERD2_PROXY_IDENTITY_DIR` 아래에 디렉터리를 만들고, ECDSA P-256 개인키를 생성하여 PKCS#8 PEM 으로 인코딩한 뒤 `key.p8`로 저장합니다.

```
func generateAndStoreKey(p string) (key *ecdsa.PrivateKey, err error) {
    key, err = tls.GenerateKey()
    if err != nil {
        return
    }
    pemb := tls.EncodePrivateKeyP8(key)
    err = os.WriteFile(p, pemb, 0600)
    return
}
```

그다음, CSR을 생성해 `csr.der`로 저장합니다.

```
func generateAndStoreCSR(p, id string, key *ecdsa.PrivateKey) ([]byte, error) {
    csr := x509.CertificateRequest{
        Subject:  pkix.Name{CommonName: id},
        DNSNames: []string{id},
    }
    csrb, err := x509.CreateCertificateRequest(rand.Reader, &csr, key)
    if err != nil {
        return nil, fmt.Errorf("failed to create CSR: %w", err)
    }
    if err := os.WriteFile(p, csrb, 0600); err != nil {
        return nil, fmt.Errorf("failed to write CSR: %w", err)
    }
    return csrb, nil
}
```

Rust 바이너리는 `TokenSource::load()`로 서비스 계정 JWT 를 읽고, 신뢰 앵커·key.p8·csr.der를 로드한 뒤 CSR을 gRPC 요청에 첨부합니다.

```
let req = tonic::Request::new(api::CertifyRequest {
  token: token.load()?,                   
  identity: name.to_string(),               
  certificate_signing_request: docs.csr_der.clone(),
});
let api::CertifyResponse { leaf_certificate, intermediate_certificates, valid_until } =
  IdentityClient::new(client).certify(req).await?.into_inner();
```

여기서 identity 값은 SPIFFE ID(spiffe://<trust-domain>/ns/<ns>/sa/<sa>)이며, 컨트롤 플레인은 이를 사용해 URI SAN이 해당 SPIFFE ID인 인증서를 발급합니다(CSR의 SAN은 무시).

## 4. 아이덴티티 중간(issuer) 인증서

중간 발급자 인증서는 `linkerd` 네임스페이스의 `linkerd-identity-issuer` 시크릿(Secret)에 저장됩니다. Identity 서비스가 CSR을 수신하면, 먼저 다음 정보를 이용해 Kubernetes API의 `authentication.k8s.io/v1/tokenreviews` 엔드포인트에 `TokenReview`를 생성해 토큰을 검증합니다.
- CSR에 포함된 ServiceAccount 토큰
- `identity.l5d.io` 오디언스 (이 오디언스 제한으로 Linkerd용으로 발급된 토큰만 허용됩니다)

검증에 실패하거나 토큰이 인증되지 않은 경우에는 즉시 실패로 처리됩니다. 반대로 초기 검증이 통과하면 API 서버가 토큰의 서명·만료 시각·발급자·대상 오디언스를 차례로 확인합니다.

그다음 Identity 서비스는 ServiceAccount 참조(system:serviceaccount:<namespace>:<name>)를 파싱해 각 세그먼트가 DNS-1123 레이블 규칙을 준수하는지 확인한 뒤, 설정된 트러스트 도메인 아래에서 SPIFFE URI를 구성합니다.


다음 단계에서 x509.Certificate 템플릿을 만듭니다.
- CSR에서 가져온 공개키
- SPIFFE URI로 설정된 SAN
- 생성 시점부터 24시간 뒤까지의 유효 기간(기본값)

그런 다음 `x509.CreateCertificate(rand.Reader, &template, issuerCert, csr.PublicKey, issuerKey)`를 사용해 인증서에 서명하고, 이를 프록시에 반환합니다.

이 과정을 확인하려면 `identity` 파드의 로그 레벨을 `debug`로 높여보세요.

```
kubectl logs -n linkerd       linkerd-identity-56d78cdd86-8c64w 
Defaulted container "identity" out of: identity, linkerd-proxy, linkerd-init (init)
time="2025-05-21T12:11:32Z" level=info msg="running version enterprise-2.17.1"
time="2025-05-21T12:11:32Z" level=info msg="starting gRPC license client" component=license-client grpc-address="linkerd-enterprise:8082"
time="2025-05-21T12:11:32Z" level=info msg="starting admin server on :9990"
time="2025-05-21T12:11:32Z" level=info msg="Using k8s client with QPS=100.00 Burst=200"
time="2025-05-21T12:11:32Z" level=info msg="POST https://10.247.0.1:443/apis/authorization.k8s.io/v1/selfsubjectaccessreviews 201 Created in 1 milliseconds"
time="2025-05-21T12:11:32Z" level=debug msg="Loaded issuer cert: -----BEGIN CERTIFICATE-----\nMIIBsjCCAVigAwIBAgIQZelMfABi9RPUkaa1fEXfIjAKBggqhkjOPQQDAjAlMSMw\nIQYDVQQDExpyb290LmxpbmtlcmQuY2x1c3Rlci5sb2NhbDAeFw0yNTA1MjExMjEx\nMDJaFw0yNjA1MjExMjExMDJaMCkxJzAlBgNVBAMTHmlkZW50aXR5LmxpbmtlcmQu\nY2x1c3Rlci5sb2NhbDBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABO52MoQ7mva8\nYPg7abR7rqO3UhE0csDoPgFKoqM54JAfQY9/8rwgKWn3AUvH9NKNNy46Nq0MmPFd\nZgz/qSX3i0WjZjBkMA4GA1UdDwEB/wQEAwIBBjASBgNVHRMBAf8ECDAGAQH/AgEA\nMB0GA1UdDgQWBBTSq+l58FRN+T4ZSwqPyX9EFJmysTAfBgNVHSMEGDAWgBQpPJRY\nnNGBgGrC7LAnIDcwXkIHVjAKBggqhkjOPQQDAgNIADBFAiA7bw59dCwkhQ9CSyUN\nLR4/U7nt2mFV519zCtvD5cJmjgIhAKhPME9EJVtN28L6ZpaYSWbnSTyih1aL/b7m\neqW0acqg\n-----END CERTIFICATE-----\n"
time="2025-05-21T12:11:32Z" level=debug msg="Issuer has been updated"
time="2025-05-21T12:11:32Z" level=info msg="starting gRPC server on :8080"
time="2025-05-21T12:11:37Z" level=debug msg="Validating token for linkerd-identity.linkerd.serviceaccount.identity.linkerd.cluster.local"
time="2025-05-21T12:11:37Z" level=info msg="POST https://10.247.0.1:443/apis/authentication.k8s.io/v1/tokenreviews 201 Created in 2 milliseconds"
time="2025-05-21T12:11:37Z" level=info msg="issued certificate for linkerd-identity.linkerd.serviceaccount.identity.linkerd.cluster.local until 2025-05-22 12:11:57 +0000 UTC: a7048ff55002e726894ad92eccfd6738fcbc72b496d58ef3071a73c866c8e311"
```

## 5. The Proxy Leaf Certificate

프록시는 발급받은 인증서를 메모리 스토어에 적재해 즉시 mTLS에 사용하며, TTL의 약 70 % 시점에서 자동 갱신합니다.

```
fn refresh_in(config: &Config, expiry: SystemTime) -> Duration {
    match expiry.duration_since(SystemTime::now()).ok().map(|d| d * 7 / 10) // 70% duration
    {
        None => config.min_refresh,
        Some(lifetime) if lifetime < config.min_refresh => config.min_refresh,
        Some(lifetime) if config.max_refresh < lifetime => config.max_refresh,
        Some(lifetime) => lifetime,
    }
}
```

이 과정을 통해 워크로드는 다운타임 없이 인증서를 회전하며, 메시 전반의 안전한 통신이 유지됩니다.
