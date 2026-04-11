# ingress-nginx

[ingress-nginx](https://github.com/kubernetes/ingress-nginx) is the reverse proxy for
BaumKubernetes, replacing nginx-proxy-manager's routing role. It terminates TLS
(via cert-manager) and routes external traffic to cluster services.

## Install via Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.ingressClassResource.default=true
```

For a bare-metal cluster without a cloud load balancer, use NodePort or
[MetalLB](https://metallb.universe.tf):

```bash
# NodePort (access via node IP)
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=80 \
  --set controller.service.nodePorts.https=443
```

## Verify

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

## Ingress pattern used in this repo

Every app's `ingress.yaml` follows this pattern:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

Update `host:` values throughout with your actual domain before deploying.
