# WSO2 API Manager Kubernetes Deployment

This repository contains Kubernetes deployment configurations for WSO2 API Manager with Envoy Gateway as the API gateway, including Control Plane and Gateway components.

## Architecture Overview

This deployment consists of three main components:

1. **Control Plane** - Central configuration management for API artifacts
2. **External Gateway** - Public API gateway for external traffic
3. **Internal Gateway** - Private API gateway for internal traffic
4. **Envoy Gateway** - Service proxy for managing and routing API traffic

## Project Structure

The deployment is organized into the following directories:

```
wso2-kube/
├── control-plane/     # Control Plane configurations
├── external-gw/       # External Gateway configurations
├── internal-gw/       # Internal Gateway configurations
├── gateway.yaml        # Main Gateway configuration
```

## Components

### Control Plane
- **Control Plane 1 & 2** - Configuration management components
- Handles API deployment, throttling policies, and analytics

### External Gateway
- Public-facing gateway for external API traffic

### Internal Gateway
- Private gateway for internal API traffic

## Deployment Instructions

### Prerequisites
- Kubernetes cluster 1.19+
- kubectl configured and connected to your cluster

### Deploying the Stack

1. Deploy the Control Plane:
```bash
kubectl apply -f control-plane/
```

2. Deploy Gateways:
```bash
# Apply External Gateway configurations
kubectl apply -f external-gw/

# Apply Internal Gateway configurations
kubectl apply -f internal-gw/
```

## Accessing the Environment

### Envoy Gateway Integration
The deployment integrates with Envoy Gateway for API traffic management:
- TLS termination at the gateway layer
- Path-based routing for different services
- Integration with WSO2 API Manager control plane

### Access Points
- **Control Plane**: `https://apim.example.com` for management interfaces
- **External Gateway**: `https://external-gw.example.com` for public APIs
- **Internal Gateway**: `https://internal-gw.example.com` for private APIs

## Monitoring and Operations

### Filebeat Integration
- External gateway includes Filebeat sidecar for log aggregation
- Centralized logging for monitoring and debugging

## Maintenance

### Scaling
- Gateways can be scaled based on traffic requirements

## Additional Resources

- [WSO2 API Manager Documentation](https://apim.docs.wso2.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
