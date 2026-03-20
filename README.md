# 🚀 AgentGateway + kagent (Kubernetes Deployment)

## 🎯 Purpose

Deploy and configure:

- agentgateway as an LLM routing layer (Helm deployment in Kubernetes)
- kagent for agent-based interaction
- Secure configuration using Secrets and ConfigMaps
- Route LLM traffic through agentgateway

---

## 🏗 Architecture

kagent → agentgateway → LLM Provider (OpenAI)

---

## ⚙️ Environment

- Local Kubernetes cluster (k3d)
- Helm
- kubectl
- Port-forward access

---

## 📦 Deployment Steps

### 1. Namespaces
kubectl create namespace agentgateway-system
kubectl create namespace kagent

---

### 2. Gateway API
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml

---

### 3. agentgateway (Helm)

helm upgrade -i \
  --namespace agentgateway-system \
  --create-namespace \
  --version v1.0.0 \
  agentgateway-crds oci://cr.agentgateway.dev/charts/agentgateway-crds

helm upgrade -i \
  -n agentgateway-system \
  agentgateway oci://cr.agentgateway.dev/charts/agentgateway \
  --version v1.0.0

---

### 4. Gateway

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
    - protocol: HTTP
      port: 80
      name: http
      allowedRoutes:
        namespaces:
          from: All

---

### 5. Secret

export OPENAI_API_KEY='<your-key>'

---

### 6. Backend

apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai:
        model: gpt-4o-mini
  policies:
    auth:
      secretRef:
        name: openai-secret

---

### 7. Route

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: agentgateway-proxy
  rules:
    - backendRefs:
        - name: openai
          group: agentgateway.dev
          kind: AgentgatewayBackend

---

### 8. Test

kubectl port-forward deployment/agentgateway-proxy -n agentgateway-system 18080:80

---

### 9. kagent

helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds -n kagent --create-namespace
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent -n kagent

---

### 10. ModelConfig

apiVersion: kagent.dev/v1alpha2
kind: ModelConfig
metadata:
  name: default-model-config
  namespace: kagent
spec:
  apiKeySecret: kagent-openai
  apiKeySecretKey: OPENAI_API_KEY
  model: gpt-4o-mini
  provider: OpenAI
  openAI:
    baseUrl: http://agentgateway-proxy.agentgateway-system.svc.cluster.local/v1

---

### 11. UI

kubectl port-forward -n kagent svc/kagent-ui 18000:8080

http://localhost:18000

---

## 📸 Screenshots

k9s:
[k9s.png]

Logs:
[logs.png]

UI:
[kagent.png]

---

## ✅ Result

- agentgateway deployed
- routing works
- kagent connected via gateway
- agent works via UI
