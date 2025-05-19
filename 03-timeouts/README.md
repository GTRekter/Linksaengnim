# Timeuts

Linkerd provides fine‑grained timeout settings to control the lifecycle of HTTP requests and TCP connections between services in your mesh. You can configure three primary timeout policies via Kubernetes service annotations:

- **timeout.linkerd.io/request:** Maximum time from when a request is sent until the first byte of the request body arrives at the server.
- **timeout.linkerd.io/response:** Maximum time from when the first byte of the response header is received until the entire response body is delivered.
- **timeout.linkerd.io/idle:** Maximum time of inactivity between data frames (both request and response) before the connection is closed.

## References
- https://linkerd.io/2.18/reference/timeouts/

# Prerequisites

- macOS/Linux/Windows with a Unix‑style shell
- k3d (v5+) for local Kubernetes clusters
- kubectl (v1.25+)
- Helm (v3+)
- Smallstep (step) CLI for certificate generation

# Tutorial

1. Create a Local Kubernetes Cluster

Use k3d and your cluster.yaml to spin up a lightweight Kubernetes cluster:

```
k3d cluster create my-linkerd-cluster \
  --kubeconfig-update-default \
  -c ./cluster.yaml
```

2. Generate Identity Certificates

Linkerd requires a trust anchor (root CA) and an issuer (intermediate CA) for mTLS identity.

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

3. Install Linkerd via Helm

```
helm repo add linkerd-edge https://helm.linkerd.io/edge
helm repo update
helm install linkerd-crds linkerd-edge/linkerd-crds \
  -n linkerd --create-namespace --set installGatewayAPI=true
helm install linkerd-control-plane \
  -n linkerd \
  --set-file identityTrustAnchorsPEM=./certificates/ca.crt \
  --set-file identity.issuer.tls.crtPEM=./certificates/issuer.crt \
  --set-file identity.issuer.tls.keyPEM=./certificates/issuer.key \
  linkerd-edge/linkerd-control-plane
```

4. Deploy the Sample Application

Assuming your app is kustomized under ./application, deploy with:

```
kubectl apply -k ./application
```

You can then validate that the traffic is going from the client's pod to server's deployment

```
kubectl exec -n simple-app client -c curl -- sh -c '\
  curl -sS --no-progress-meter \
    -X POST \
    -H "Content-Type: application/json" \
    -o /dev/null \
    -w "HTTPSTATUS:%{http_code}\n" \
    http://server.simple-app.svc.cluster.local/post \
'
```

5. Timeout Scenarios

Below are examples of how to annotate your server service and test each timeout policy.

## Request Timeout (timeout.linkerd.io/request)

Limit the time to upload the entire request body by setting the following annotation to the server's service.

```
kubectl annotate svc server \
  -n simple-app \
  timeout.linkerd.io/request=1s \
  timeout.linkerd.io/response=1h \
  timeout.linkerd.io/idle=1h \
  --overwrite
```

Throttle the upload to take longer than 1s (100 KB at 20 KB/s):

```
kubectl exec -n simple-app client -c curl -- sh -c '\
  yes a | head -c 100000 > /tmp/payload.json && \
  curl -sS --no-progress-meter \
    --limit-rate 20K \
    -X POST \
    -H "Content-Type: application/json" \
    --data-binary @/tmp/payload.json \
    -o /dev/null \
    -w "HTTPSTATUS:%{http_code}\n" \
    http://server.simple-app.svc.cluster.local/post \
'
```

## Response Timeout (timeout.linkerd.io/response)

Limit the time to receive the full response by setting the following annotation to the server's service.

```
kubectl annotate svc server \
  -n simple-app \
  timeout.linkerd.io/request=1h \
  timeout.linkerd.io/response=1s \
  timeout.linkerd.io/idle=1h \
  --overwrite
```

Use HTTPBin’s /delay/5 endpoint to delay the response >5s:

```
kubectl exec -n simple-app client -c curl -- sh -c '\
  curl -sS --no-progress-meter \
    -X POST \
    -H "Content-Type: application/json" \
    -o /dev/null \
    -w "HTTPSTATUS:%{http_code}\n" \
    http://server.simple-app.svc.cluster.local/delay/5 \
'
```