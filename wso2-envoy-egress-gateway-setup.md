# WSO2 API Manager Egress Gateway Setup (Cilium Envoy Data Plane)

This documentation tracks the deployment and configuration of the enterprise egress gateway for WSO2 API Manager using Cilium's L7 engine, complete with Layer 2 hardware address announcements and terminated TLS mapping.

---

## 1. Prerequisites

Before beginning the installation, ensure you have the following:

* Kubernetes cluster 1.19+
* kubectl configured and connected to your cluster
* Helm 3.x installed

---

## 2. Envoy Proxy Installation

### Install Gateway API CRDs

First, install the Gateway API Custom Resource Definitions:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

### Install Envoy Gateway

Install Envoy Gateway using Helm:

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.8.0 \
  -n envoy-gateway-system \
  --create-namespace
```

Verify the installation:

```bash
kubectl get pods -n envoy-gateway-system
```

---

## 3. Architecture Overview

The infrastructure is broken down into distinct conceptual layers managed by Kubernetes custom resources:

* **Layer 2 Infrastructure:** Maps IP allocation pools to physical hypervisor/node network interfaces using ARP tracking.
* **Control/Proxy Plane:** Managed inside the `envoy-gateway-system` namespace. Handles the external virtual IP assignment and TLS edge termination.
* **Application Route Plane:** Lives in the workload namespace. Evaluates incoming host headers and routes backend traffic securely over upstream ports.

---

## 4. Infrastructure Setup (Layer 2 & IPAM)

To allow the Cilium Envoy proxies to communicate with your local network infrastructure, we define a dedicated IPAM pool and bind it to a local hardware network interface card regex pattern.

Save this configuration as `cilium-l2-infra.yaml`:

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumLoadBalancerIPPool
metadata:
  name: local-egress-pool
  namespace: kube-system
spec:
  blocks:
    - cidr: "192.168.50.200/29" # Reserves IPs from .200 through .207
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: local-l2-policy
  namespace: kube-system
spec:
  serviceSelector:
    matchLabels: {} 
  nodeSelector:
    matchLabels: {} 
  interfaces:
    - "^(eth|ens|enp|br).*$" # Broad regex matching eth0, ens18, enp3s0, or network bridges
  loadBalancerIPs: true
```

Apply the configuration:
```bash
kubectl apply -f cilium-l2-infra.yaml
```

---

## 5. Edge Gateway & Route Architecture

The explicit architecture maps edge termination and binds the backend microservices cleanly across decoupled structural namespaces.

Save the configuration below as `wso2-gateway-services.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: wso2-gateway
  namespace: envoy-gateway-system
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      hostname: "*.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - group: ""
            kind: Secret
            name: wso2-ingress-cert
            namespace: envoy-gateway-system
      allowedRoutes:
        namespaces:
          from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: apim-route
  namespace: wso2-cp
spec:
  parentRefs:
    - name: wso2-gateway
      namespace: envoy-gateway-system
      sectionName: https
  hostnames:
    - "apim.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: apim-wso2am-acp-service
          port: 9443
---
apiVersion: gateway.networking.k8s.io/v1
kind: BackendTLSPolicy
metadata:
  name: wso2-backend-tls
  namespace: wso2-cp
spec:
  targetRefs:
    - group: ""
      kind: Service
      name: apim-wso2am-acp-service
  validation:
    hostname: "localhost"
    caCertificateRefs:
      - group: ""
        kind: ConfigMap
        name: wso2-ca-cert
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: internal-gw-route
  namespace: wso2-internal-gw
spec:
  parentRefs:
    - name: wso2-gateway
      namespace: envoy-gateway-system
      sectionName: https
  hostnames:
    - "internal-gw.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: apim-gw-wso2am-universal-gw-service
          port: 8243
---
apiVersion: gateway.networking.k8s.io/v1
kind: BackendTLSPolicy
metadata:
  name: internal-gw-backend-tls
  namespace: wso2-internal-gw
spec:
  targetRefs:
    - group: ""
      kind: Service
      name: apim-gw-wso2am-universal-gw-service
  validation:
    hostname: "localhost"
    caCertificateRefs:
      - group: ""
        kind: ConfigMap
        name: wso2-ca-cert
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: external-gw-route
  namespace: wso2-external-gw
spec:
  parentRefs:
    - name: wso2-gateway
      namespace: envoy-gateway-system
      sectionName: https
  hostnames:
    - "external-gw.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: apim-gw-wso2am-universal-gw-service
          port: 8243
---
apiVersion: gateway.networking.k8s.io/v1
kind: BackendTLSPolicy
metadata:
  name: external-gw-backend-tls
  namespace: wso2-external-gw
spec:
  targetRefs:
    - group: ""
      kind: Service
      name: apim-gw-wso2am-universal-gw-service
  validation:
    hostname: "localhost"
    caCertificateRefs:
      - group: ""
        kind: ConfigMap
        name: wso2-ca-cert
```

Apply the unified data configurations:

```bash
kubectl apply -f wso2-gateway-services.yaml
```

---

## 6. Security & Secret Provisions

The edge proxy requires the target SSL private and public keys to exist natively inside its tracking namespace to compute the TLS handshake successfully.

Create the Edge TLS Secret:

```bash
kubectl create secret tls wso2-ingress-cert \
  --namespace envoy-gateway-system \
  --cert=/path/to/your/tls.crt \
  --key=/path/to/your/tls.key
```

Create the CA Certificate ConfigMap for internal gateway:

```bash
kubectl create configmap wso2-ca-cert -n wso2-cp --from-file=ca.crt=wso2-actual.crt
kubectl create configmap wso2-ca-cert -n wso2-internal-gw --from-file=ca.crt=wso2-actual.crt
kubectl create configmap wso2-ca-cert -n wso2-external-gw --from-file=ca.crt=wso2-actual.crt
```

---

## 7. Verification & Operations Runbook

### Check Resource Bindings

Confirm that Cilium has compiled the objects and injected the successful controller status blocks:

```bash
kubectl get gatewayclass,gateway -n envoy-gateway-system
kubectl get httproute -n wso2-cp
kubectl get httproute -n wso2-internal-gw
kubectl get httproute -n wso2-external-gw
```

### Verification Commands

```bash
# Check gateway class status
kubectl get gatewayclass -n envoy-gateway-system

# Check gateway status
kubectl get gateway -n envoy-gateway-system

# Describe HTTP route
kubectl describe httproute -n wso2-cp
kubectl describe httproute -n wso2-internal-gw
kubectl describe httproute -n wso2-external-gw

# Check Envoy Gateway pods
kubectl get pods -n envoy-gateway-system

# Check configmap creation
kubectl get configmap wso2-ca-cert -n wso2-cp
kubectl get configmap wso2-ca-cert -n wso2-internal-gw
kubectl get configmap wso2-ca-cert -n wso2-external-gw
```

