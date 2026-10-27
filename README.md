### CREATE A KIND CLUSTER WITH GATEWAY API

# Create a cluster kind-config.yaml 
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker

# Kind create cluster
kind create cluster --name lab --config ./kind-config.yaml 

# Apply docker image to kind-cluster
docker build -t my-custom-image:unique-tag ./my-image-dir
kind load docker-image my-custom-image:unique-tag --name <cluster-name>

# Install Gateway API CRDs
# These Custom Resource Definitions tell Kubernetes what Gateway, HTTPRoute etc. are.
# Without them, kubectl won't recognize these resource types.
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

# Install cloud-provider-kind
# https://github.com/kubernetes-sigs/cloud-provider-kind
#
# In a real cloud environment (AWS, Azure, GCP), creating a Service of type LoadBalancer
# automatically provisions a cloud load balancer with a public IP. Kubernetes doesn't do
# this on its own — it delegates to the cloud provider.
#
# kind runs locally with no cloud behind it, so LoadBalancer Services stay stuck at
# `<pending>` forever and Gateway API has no controller to handle traffic.
#
# cloud-provider-kind solves this by simulating a cloud provider for kind. It runs as a
# separate process, monitors the cluster, and when it detects a LoadBalancer Service or
# a Gateway resource, it creates a Docker container that exposes the service on your host.

apt install -y golang
go install sigs.k8s.io/cloud-provider-kind@latest
sudo install ~/go/bin/cloud-provider-kind /usr/local/bin

# Run cloud-provider-kind in a separate terminal (must stay running)
# --gateway-channel standard tells it to support Gateway API using
# the standard release channel (stable features only, no experimental)
cloud-provider-kind --gateway-channel standard

# Apply manifests
kubectl apply -f deployment.yaml
kubectl apply -f gateway.yaml
kubectl apply -f httproute.yaml

# Verify everything is running
kubectl get all -n prj-go-server
kubectl get gateway -n prj-go-server       # should show ADDRESS and Programmed: True
kubectl get httproute -n prj-go-server     # should show the route

# Get the Gateway external IP
kubectl get gateway -n prj-go-server -o jsonpath='{.items[0].status.addresses[0].value}'

# Add to /etc/hosts (replace IP with the one from above)
echo "172.18.0.5 kind.local" >> /etc/hosts

# Test
curl kind.local

### REFERENCES
# https://gateway-api.sigs.k8s.io/
# https://github.com/kubernetes-sigs/cloud-provider-kind
# https://kind.sigs.k8s.io/docs/user/quick-start/

### GUIDES
https://medium.com/@standardloop/nov-2025-using-k8s-gateway-api-with-kind-and-cloud-provider-kind-3a724b926780#48ce
https://devopscube.com/kubernetes-gateway-api/
https://github.com/arapulido/gateway-api-examples
