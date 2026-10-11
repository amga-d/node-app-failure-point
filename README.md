# LAB: Create a ReplicaSet for a Node.js app that includes a failure point

# Requirements
1. A Kubernetes cluster (single-node)
2. Docker installed on the host
3. The Kubernetes cluster must be reachable from the host

# Environment
1. Minikube using the kvm2 driver with a custom network such as 192.168.39.0/24
2. A local registry running on the host, for example 0.0.0.0:5000

# Steps

## 1. Build the image and push it to the registry

### 1. Deploy the local registry
```bash
docker run --rm -d --name registry -p 5000:5000 registry:2
```

### 2. Build the image on the host
```bash
docker build -t node-app:v1 .
```

### 3. Retag the image and push it to the registry
```bash
docker tag node-app:v1 localhost:5000/node-app:v1
docker push localhost:5000/node-app:v1
```

### Note
You must configure the registry in your cluster as an insecure registry. If you are using containerd as the container runtime, complete the following steps.

**SSH into the node**
```bash
sudo mkdir -p /etc/containerd/certs.d/192.168.39.1:5000

sudo tee /etc/containerd/certs.d/192.168.39.1:5000/hosts.toml <<'EOF'
server = "http://192.168.39.1:5000"

[host."http://192.168.39.1:5000"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
EOF
```

Replace the IP and port with your host's registry address: `<host-ip:registry-port>`.

## 2. Apply the ReplicaSet and Service

### 1. Apply the ReplicaSet
```bash
kubectl apply -f ReplicaSet.yaml
```

To expose the pods, create a Service in front of them. Since the host can reach the VM directly, use a `NodePort`.

### 2. Apply the Service (NodePort)
```bash
kubectl apply -f Service.yaml
```

The Service listens on the pod port `targetPort: 3000`, exposes the three pods on the cluster network using `port: 80`, and opens a port on the node using `nodePort: 30008`.

### 3. Check the endpoint
The endpoint output should list three pod IPs:
```bash
kubectl get endpoints node-app
```

Example output:
```bash
NAME       ENDPOINTS                                    AGE
node-app   10.244.0.40:3000,10.244.0.41:3000,10.244.0.42:3000   57m
```

## 3. Send requests to the app to trigger the failure

### Note
You may open two terminal windows or use `tmux` to split the screen into two panes.

### 1. Terminal 1: Watch the pod status
```bash
kubectl get pods -l app=failure-point -o wide --watch
```

Notice that `-l` filters the output to only pods with the label `app=failure-point`.

### 2. Terminal 2: Send a request to the `/failure` endpoint to trigger the container to stop
```bash
# curl http://<vm-ip>:<node-port>/failure
curl http://192.168.39.131:30008/failure
```

Each time you send the request, a random pod will briefly enter the `Error` state before restarting.



