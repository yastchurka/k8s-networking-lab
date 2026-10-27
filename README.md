# k8s-networking-lab

CKA prep - Kubernetes networking lab on kind — Services, Gateway API, Network Policies, and monitoring with Prometheus and Grafana.

## Prerequisites

- Docker
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
- kubectl
- Go (for cloud-provider-kind)

## Setup

### 1. Create the cluster

```bash
kind create cluster --name lab --config kind-config.yaml
```

### 2. Install Gateway API CRDs

These Custom Resource Definitions tell Kubernetes what Gateway, HTTPRoute etc. are. Without them, kubectl won't recognize these resource types.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

### 3. Install and run cloud-provider-kind

In a real cloud environment (AWS, Azure, GCP), creating a Service of type LoadBalancer automatically provisions a cloud load balancer with a public IP. Kubernetes doesn't do this on its own — it delegates to the cloud provider.

kind runs locally with no cloud behind it, so LoadBalancer Services stay stuck at `<pending>` forever and Gateway API has no controller to handle traffic.

cloud-provider-kind solves this by simulating a cloud provider for kind. It runs as a separate process, monitors the cluster, and when it detects a LoadBalancer Service or a Gateway resource, it creates a Docker container that exposes the service on your host.

```bash
go install sigs.k8s.io/cloud-provider-kind@latest
sudo install ~/go/bin/cloud-provider-kind /usr/local/bin
```

Run in a separate terminal (must stay running):

```bash
cloud-provider-kind --gateway-channel standard
```

### 4. Build and load the application image

```bash
docker build -t go-server:1.0 ./app
kind load docker-image go-server:1.0 --name lab
```

`kind load` copies the image from Docker on your host into every node of the kind cluster. Without this, pods fail with `ErrImageNeverPull` because the image only exists on the host, not inside the cluster nodes.

### 5. Deploy

```bash
kubectl apply -f deployment.yaml
kubectl apply -f gateway.yaml
kubectl apply -f httproute.yaml
```

### 6. Verify

```bash
kubectl get all -n prj-go-server
kubectl get gateway -n prj-go-server
kubectl get httproute -n prj-go-server
```

Gateway should show an `ADDRESS` and `Programmed: True`.

### 7. Test

Get the Gateway IP:

```bash
kubectl get gateway -n prj-go-server -o jsonpath='{.items[0].status.addresses[0].value}'
```

Add it to `/etc/hosts`:

```
172.18.0.5 kind.local
```

```bash
curl kind.local
```

## Project structure
<img width="1384" height="1946" alt="image" src="https://github.com/user-attachments/assets/266ffbbd-e089-4ef5-a3ba-8852673ffff4" />

