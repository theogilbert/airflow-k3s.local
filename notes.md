# What has been done to setup k3s

## Setup playground dir

```bash
cd ~/projects
mkdir kube
cd kube
export PATH=$PATH:~/projects/kube
```

## Download k3s

```bash
wget https://github.com/k3s-io/k3s/releases/download/v1.37.1%2Bk3s1/k3s
chmod +x k3s
```

## Download kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
ln -s kubectl k
```

## Run server

```bash
sudo k3s server
```

## Trust cluster networks in firewalld

```bash
# Fedora's firewalld rejects forwarded pod-to-pod traffic on the cni0 bridge (traffic via a service IP still works).
# Traefik sends requests straight to pod IPs, so ingresses returned 502 Bad Gateway:
#   from the traefik pod: wget http://<pod-ip>:8080/  ->  Host is unreachable
sudo firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16   # pods
sudo firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16   # services
sudo firewall-cmd --reload
```

## Copy root kubeconfig

```bash
sudo cat /etc/rancher/k3s/k3s.yaml > rootkubeconfig.yml
export KUBECONFIG=rootkubeconfig.yml
```

## Create airflow-helm kubeconfig

```bash
# 1. Create a key and have the cluster sign a certificate
openssl genrsa -out airflow-helm.key 2048
openssl req -new -key airflow-helm.key -out airflow-helm.csr -subj "/CN=airflow-helm/O=devs"
#   CN = username, O = group (optional; you can bind permissions to groups too)

cat <<EOF | kubectl apply -f - 
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: airflow-helm
spec:
  request: $(base64 -w0 < airflow-helm.csr)
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 7776000   # 90 days
  usages: ["client auth"]
EOF

kubectl certificate approve airflow-helm
kubectl get csr airflow-helm -o jsonpath='{.status.certificate}' | base64 -d > airflow-helm.crt

# 2. Grant permissions with RBAC

kubectl create namespace airflow

# Grant full-admin rights within the airflow namespace
kubectl create rolebinding airflow-helm-admin --clusterrole=admin --user=airflow-helm -n airflow

# 3. Build the kubeconfig

# extract the cluster CA from the admin kubeconfig
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > ca.crt

K=airflow-helm.kubeconfig
kubectl config --kubeconfig=$K set-cluster k3s --server=https://localhost:6443 --certificate-authority=ca.crt --embed-certs
kubectl config --kubeconfig=$K set-credentials airflow-helm --client-certificate=airflow-helm.crt --client-key=airflow-helm.key --embed-certs
kubectl config --kubeconfig=$K set-context airflow-helm@k3s --cluster=k3s --user=airflow-helm --namespace=airflow
kubectl config --kubeconfig=$K use-context airflow-helm@k3s
# Use 127.0.0.1 as the server address if airflow-helm works on the node itself, or the node's IP if she connects from elsewhere.

# 4. Test it

export KUBECONFIG=airflow-helm.kubeconfig
kubectl get pods                  # works (airflow namespace)
```

## Install Helm

```bash
wget https://get.helm.sh/helm-v4.3.0-linux-amd64.tar.gz
tar xzf helm-v4.3.0-linux-amd64.tar.gz
mv linux-amd64/helm .
chmod +x helm
```

## Create airflow namespace

```bash
helm repo add apache-airflow https://airflow.apache.org
helm install airflow apache-airflow/airflow -n airflow
```

## Block NET_RAW in the airflow namespace

```bash
# k3s docs warning: flannel uses a bridge (L2) network, so a pod with the NET_RAW capability can ARP-spoof other pods.
# The airflow chart's own pods already drop ALL capabilities, but pods launched by DAGs (KubernetesPodOperator)
# or created by airflow-helm get containerd's defaults, which include NET_RAW.
# PSA "restricted" would fix it but also forbids root containers, so use a small admission policy instead.

# Run with the root kubeconfig.
export KUBECONFIG=rootkubeconfig.yml

# Reject pods in airflow whose containers don't drop NET_RAW (or ALL); root is still allowed.
kubectl apply -f drop-net-raw.yaml

# Also forbid privileged containers and adding NET_RAW back (baseline allows root).
kubectl label ns airflow pod-security.kubernetes.io/enforce=baseline

# Check, empty output = the policy's CEL compiled fine:
#   kubectl get validatingadmissionpolicy drop-net-raw -o jsonpath='{.status.typeChecking}'
# Only affects new pods. Pods must now set:
#   securityContext: {capabilities: {drop: [NET_RAW]}}
```

## Configure airflow

```bash
helm upgrade airflow apache-airflow/airflow -n airflow --reuse-values -f override.yml
```
