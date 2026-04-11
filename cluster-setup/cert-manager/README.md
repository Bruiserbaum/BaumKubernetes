# cert-manager

[cert-manager](https://cert-manager.io) automates TLS certificate provisioning from
Let's Encrypt. Every app Ingress in BaumKubernetes uses a `Certificate` resource or
the `cert-manager.io/cluster-issuer` annotation.

## Install

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

## Create a ClusterIssuer

Replace `your-email@example.com` with a real address (Let's Encrypt notifications).

```bash
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

For local / internal-only clusters without public DNS, use a **staging** issuer
or a self-signed certificate instead:

```bash
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
EOF
```

## Usage in Ingress resources

All app Ingress resources in this repo include the annotation:

```yaml
annotations:
  cert-manager.io/cluster-issuer: letsencrypt-prod
```

And a `tls:` block referencing a secret name where the certificate will be stored.
cert-manager handles issuance and renewal automatically.
