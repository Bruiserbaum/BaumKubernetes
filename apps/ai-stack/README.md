# AI Stack

**Local AI / LLM stack** — Ollama model server, LibreChat multi-model UI,
AnythingLLM document RAG, and n8n workflow automation.

**Namespace:** `ai-stack`

| Component | Image | Purpose | Source |
|-----------|-------|---------|--------|
| Ollama | `ollama/ollama:latest` | LLM inference server | [ollama/ollama](https://github.com/ollama/ollama) |
| LibreChat | `ghcr.io/danny-avila/librechat:latest` | Multi-model chat UI | [danny-avila/LibreChat](https://github.com/danny-avila/LibreChat) |
| AnythingLLM | `mintplexlabs/anythingllm:latest` | Document RAG + chat | [Mintplex-Labs/anything-llm](https://github.com/Mintplex-Labs/anything-llm) |
| n8n | `n8nio/n8n:latest` | Workflow automation | [n8n-io/n8n](https://github.com/n8n-io/n8n) |
| MongoDB | `mongo:7` | LibreChat database | [mongodb](https://hub.docker.com/_/mongo) |

---

## Secrets required

```bash
kubectl create namespace ai-stack

kubectl create secret generic librechat-secret \
  --namespace ai-stack \
  --from-literal=JWT_SECRET="$(openssl rand -hex 32)" \
  --from-literal=CREDS_KEY="$(openssl rand -hex 32)" \
  --from-literal=SESSION_SECRET="$(openssl rand -hex 32)"
```

Optional OpenAI fallback key:
```bash
kubectl patch secret librechat-secret -n ai-stack \
  --type merge --patch '{"stringData":{"OPENAI_API_KEY":"sk-..."}}'
```

---

## Deploy

```bash
# 1. Create secrets
# 2. Edit ingress.yaml — set hosts for librechat, anythingllm, n8n
kubectl apply -k apps/ai-stack/
```

## Deploy via Portainer

> Create the namespace and secrets first using the `kubectl` commands in **Secrets required** above.

1. In Portainer: **Kubernetes** → **Manifests** → **Deploy**
2. Select **Repository**
3. Enter:
   - Repository URL: `https://github.com/Bruiserbaum/BaumKubernetes`
   - Compose path: `apps/ai-stack/kustomization.yaml`
   - Namespace: `ai-stack`
4. Click **Deploy**

---

## Pull a model after Ollama starts

```bash
kubectl exec -n ai-stack deployment/ollama -- ollama pull llama3.2
kubectl exec -n ai-stack deployment/ollama -- ollama pull nomic-embed-text
```

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `ollama-pvc` | `local-path-fast` | 50Gi | Downloaded models |
| `mongodb-pvc` | `local-path-fast` | 10Gi | LibreChat conversations |
| `n8n-pvc` | `local-path-fast` | 5Gi | n8n workflows |
| `anythingllm-pvc` | `local-path-fast` | 20Gi | Documents + vector store |

---

## Exposed services

| Service | URL |
|---------|-----|
| LibreChat | `https://chat.example.com` |
| AnythingLLM | `https://anythingllm.example.com` |
| n8n | `https://n8n.example.com` |
| Ollama API | ClusterIP only (port 11434) |

---

## SSO with Authentik (LibreChat)

In Authentik, create an OpenID Connect provider, then configure LibreChat:

```bash
kubectl patch configmap librechat-config -n ai-stack --patch '
data:
  OPENID_ISSUER: "https://authentik.example.com/application/o/librechat/"
  OPENID_CLIENT_ID: "librechat"
  OPENID_SCOPE: "openid profile email"
'
kubectl create secret generic librechat-oidc -n ai-stack \
  --from-literal=OPENID_CLIENT_SECRET="<client secret>"
```
