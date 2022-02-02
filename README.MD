***BEFORE START: Make sure that swap is off***
```
sudo swapoff -a
```
OR edit `/etc/fstab` and comment any swap entries then reboot your server

# Docker
```
sudo apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release -y

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg


echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

sudo groupadd docker
sudo usermod -aG docker $USER

```

### docker cgroup config
```
sudo vim /etc/docker/daemon.json
```
Add this config:
```
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

# Kubernetes

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

## Kubeadm, Kubelete, and Kubectl
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```


### Initialize kubeadm
```
sudo kubeadm init -v=6 --pod-network-cidr=192.168.0.0/16 

```

**As Normal user Or Root choose one method**
```
# as normatl user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively, if you are the root user, you can run:
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```

## Install  Calico network plugin
```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

After adding manifest files, follow these steps:
1. Confirm that all of the pods are running with the following command. (Wait until each pod has the STATUS of Running.)
    ```
    watch kubectl get pods -n calico-system
    ```

1. Remove the taints on the master so that you can schedule pods on it.
    ```
    kubectl taint nodes --all node-role.kubernetes.io/master-
    ```

1. Confirm that you have a node in your cluster with the following command.
    ```
    kubectl get nodes -o wide
    ```

1. Install calicoctl 
    ```
    curl -o calicoctl -O -L  "https://github.com/projectcalico/calicoctl/releases/download/v3.20.2/calicoctl" 

    sudo chmod +x calicoctl

    sudo mv calicoctl /usr/bin/calicoctl
    ```

## Enable bash completion
```
sudo -i

sudo kubectl completion bash >> /etc/bash_completion.d/kubectl

kubeadm completion bash >> /etc/bash_completion.d/kubeadm

exit

```
Logout and login again to enable auto completion

## Create new join token
```
kubeadm token list

# create new token
kubeadm token create

openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

```

## Dashboard
To install kubernetes dashboard, execute these commands as normal user
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```

Get pods of kubernetes-dashboard namespace
```
kubectl get pods -n kubernetes-dashboard
```

Copy the name of kubernetes-dashboard pod and replace in the following command:
```
kubectl port-forward --namespace=kubernetes-dashboard POD_NAME_HERE --address=0.0.0.0 8443:8443
```

**Find Dashboard port**

```
kubectl describe --namespace=kubernetes-dashboard services
```
Look at the Endpoints of dashboard service to find the service port and in `kubectl port-forward --namespace=kubernetes-dashboard kubernetes-dashboard-67484c44f6-6rrsb --address=0.0.0.0 8443:8443` command replace the `8443:8443` with the correct port. Also replace the port in the link below.

Visit [dashboard](https://127.0.0.1:8443)

### Generate bearer token for dashboard
```
cat <<EOF>> dashboard-adminuser.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

kubectl apply -f dashboard-adminuser.yaml
```

Find the generated token using these commands:
```
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"

# OR

kubectl describe secrets admin-user -n kubernetes-dashboard # Copy token
```

# Helm
```
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm -y

sudo -i
helm completion bash  >> /etc/bash_completion.d/helm
exit

```
