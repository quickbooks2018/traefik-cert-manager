# Traefik Cert Manager

- 1 Cert Manager Installation

- cert-manager-values.yaml
```yaml
installCRDs: false
replicaCount: 3
extraArgs:
  - --dns01-recursive-nameservers=1.1.1.1:53,9.9.9.9:53
  - --dns01-recursive-nameservers-only
podDnsPolicy: None
podDnsConfig:
  nameservers:
    - "1.1.1.1"
    - "9.9.9.9"
```
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm repo ls
helm search repo jetstack
helm search repo jetstack/cert-manager --versions
helm show values jetstack/cert-manager --version v1.14.4
helm show values jetstack/cert-manager --version v1.14.4 > cert-manager-values.yaml
helm upgrade --install cert-manager jetstack/cert-manager --version v1.14.4 --namespace cert-manager --set installCRDs=true --create-namespace --values cert-manager-values.yaml --wait 
```

- 2 secret-cf-token.yaml
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  cloudflare-token: eqyeYLqKcFqyOEN-JpBCCe_dZdcEFDv65zgmWDcI
```

- 3 Letsencrypt-Staging-Issuer.yaml
```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      - dns01:
          cloudflare:
            email: you@example.com
            apiTokenSecretRef:
              name: cloudflare-token-secret
              key: cloudflare-token
        selector:
          dnsZones:
            - "example.com"
```

- 4 Letsencrypt-Production-Issuer.yaml
```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - dns01:
          cloudflare:
            email: you@example.com
            apiTokenSecretRef:
              name: cloudflare-token-secret
              key: cloudflare-token
        selector:
          dnsZones:
            - "example.com"
```

- verify 4
```bash
kubectl get clusterissuer letsencrypt-production -o wide 
```

- 5 Staging-Certificate.yaml
```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: local-example-com
  namespace: default
spec:
  secretName: local-example-com-staging-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: "*.local.example.com"
  dnsNames:
  - "local.example.com"
  - "*.local.example.com"
```

- 6 Production-Certificate.yaml
```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: local-example-com
  namespace: default
spec:
  secretName: local-example-com-tls
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  commonName: "*.local.example.com"
  dnsNames:
    - "*.saqlainmushtaq.com"
```

- verify 6
```bash
kubectl get certificate -n default -o wide
kubectl describe certificate local-saqlainmustaq-com -n default
k get secrets -n default
```

- 7 traefik Installation
- -traefik-values.yaml
```yaml
deployment:
  kind: Deployment
  podAnnotations:
    prometheus.io/port: "9100"
    prometheus.io/scrape: "true"
  podLabels:
    gke: traefik
  replicas: 3
ingressClass:
  enabled: true
  fallbackApiVersion: ""
  isDefaultClass: false
  name: traefik-external
ingressRoute:
  dashboard:
    enabled: false
metrics:
  prometheus:
    addEntryPointsLabels: false
    addServicesLabels: false
    entryPoint: metrics
nodeSelector:
  cloud.google.com/gke-container-runtime: containerd
ports:
  metrics:
    exposedPort: 9100
  traefik:
    exposedPort: 9000
  web:
    redirectTo: websecure
    expose: true
    exposedPort: 80
    port: 80
  websecure:
    expose: true
    exposedPort:
    # https://www.cloudflare.com/th-th/ips/
    forwardedHeaders:
      trustedIPs:
        - 103.21.244.0/22
        - 103.22.200.0/22
        - 103.31.4.0/22
        - 104.16.0.0/13
        - 104.24.0.0/14
        - 108.162.192.0/18
        - 131.0.72.0/22
        - 141.101.64.0/18
        - 162.158.0.0/15
        - 172.64.0.0/13
        - 173.245.48.0/20
        - 188.114.96.0/20
        - 190.93.240.0/20
        - 197.234.240.0/22
        - 198.41.128.0/17
        - 2400:cb00::/32
        - 2606:4700::/32
        - 2803:f800::/32
        - 2405:b500::/32
        - 2405:8100::/32
        - 2a06:98c0::/29
        - 2c0f:f248::/32
    port: 443
providers:
  kubernetesCRD:
    ingressClass: traefik-external
    allowExternalNameServices: true
    enabled: true
  kubernetesIngress:
    allowExternalNameServices: true
    enabled: true
    ingressEndpoint:
      publishedService: traefik/traefik
resources:
  limits:
    cpu: "2"
    memory: 8Gi
  requests:
    cpu: 100m
    memory: 128Mi
securityContext:
  capabilities:
    add:
      - NET_BIND_SERVICE
    drop:
      - ALL
  readOnlyRootFilesystem: true
  runAsGroup: 0
  runAsNonRoot: false
  runAsUser: 0
service:
  annotations:
    cloud.google.com/neg: '{"ingress":true}'
    networking.gke.io/load-balancer-type: External
  enabled: true
  type: LoadBalancer

rbac:
  enabled: true

globalArguments:
  - --api.insecure=true
  - --global.sendanonymoususage=false
  - --global.checknewversion=false

additionalArguments:
  - --global.checknewversion=false
  - --api=true
  - --api.dashboard=true
  - --accesslog=true
  - --accesslog.format=json
  - --accesslog.fields.defaultmode=keep
  - --accesslog.fields.headers.defaultmode=drop
  - --log.level=info
  - --entrypoints.web.address=:80
  - --entrypoints.websecure.address=:443
  - --entryPoints.traefik.address=:9000
  - --entryPoints.metrics.address=:9100
  - --entrypoints.web.http.redirections.entryPoint.to=websecure
  - --entrypoints.web.http.redirections.entryPoint.scheme=https
  - --entrypoints.web.http.redirections.entrypoint.permanent=true
  - --ping
  - --providers.kubernetesingress
  - --providers.kubernetescrd
  - --metrics.prometheus=true
  - --metrics.prometheus.entryPoint=metrics
  - --serversTransport.insecureSkipVerify=true 
```
```bash
helm repo ls
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
helm search repo traefik
helm show values traefik/traefik
helm search repo traefik/traefik --versions
helm show chart traefik/traefik --version 24.0.0
helm show readme traefik/traefik --version 24.0.0
helm show values traefik/traefik --version 24.0.0
helm show all traefik/traefik --version 24.0.0 
helm -n traefik upgrade --install traefik --create-namespace traefik/traefik --version 24.0.0 --values=traefik-values.yaml --wait

# For Internal
helm -n traefik-internal upgrade --install traefik-internal --create-namespace traefik/traefik --version 24.0.0 --values=traefik-internal-values.yaml --wait
```

- 8 Setup VPN
```link
https://www.youtube.com/watch?v=Kfd6oKzUgbE
https://github.com/quickbooks2018/aws/blob/master/pritunl/pritunl.sh

# allow on your ip
80
443
8443 0.0.0.0/0

username: pritunl
password: ---> run this command (docker exec -it pritunl pritunl default-password)

Note: 8443 will not work, you need start the server(application) on 1194/tcp 

Note: After attaching the organization to the server click ---> start server
```