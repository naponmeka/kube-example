# kube-example
This project contains tutorial for setting up kubernetes cluster, dashboard, NGINX ingress controller.
## Setting up kubernetes cluster using kubeadm
### Reference
```
https://kubernetes.io/docs/setup/independent/install-kubeadm/
```
### Prerequisites
You'll need at least 2 vps server (in this tutorial I'm going to use Ubuntu)
```
Operating system:
- Ubuntu 16.04+
- Debian 9
- CentOS 7
- RHEL 7
- Fedora 25/26 (best-effort)
- HypriotOS v1.0.1+
- Container Linux (tested with
1800.6.0)
RAM:
2 GB or more
```
### Installing docker
https://docs.docker.com/install/linux/docker-ce/ubuntu/
Installing docker-ce
```
[Both Master and worker]
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce
```
Verify that Docker CE is installed correctly
```
docker ps
docker run hello-world
```
### Installing kubeadm, kubelet and kubectl
https://kubernetes.io/docs/setup/independent/install-kubeadm/
```
[Both Master and Worker]
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```
### Creating a single master cluster with kubeadm
https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
In this tutorial I'm going to use "Calico" as Kubernetes pod networks

Initializing cluster with kubeadm
```
[ Master ]
kubeadm init --pod-network-cidr=192.168.0.0/16
Result:
    kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

Making kubectl work
```
[ Master (non-root) ]
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
[ Master using root ]
export KUBECONFIG=/etc/kubernetes/admin.conf
```
Installing a pod network add-on (Calico)
```
[ Master ]
kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```
Verifying that the master is up and running
```
[ Master ]
kubectl get nodes
result:
NAME                         STATUS   ROLES    AGE     VERSION
ubuntu-s-1vcpu-2gb-sgp1-01   Ready    master   9m40s   v1.12.1
```
Joing worker with master
```
[Worker]
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>

```
Verifying that the worker is up and joined with the cluster
```
[ Master ]
kubectl get nodes
result:
NAME                         STATUS   ROLES    AGE   VERSION
ubuntu-s-1vcpu-2gb-sgp1-01   Ready    master   13m   v1.12.1
ubuntu-s-1vcpu-2gb-sgp1-02   Ready    <none>   38s   v1.12.1

```

## Setting up kubernetes dashboard
### Reference
```
https://orchestration.io/2018/03/15/configuring-kubernetes-dashboard-to-work-with-kubeadm-provisioned-clusters/
```
### Installing the Dashboard (kubeadm provisioned clusters)
```
[ Master ]
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```
verifying that the dashboard is up and running
(look for kubernetes-dashboard-)
```
[ Master ]
kubectl get pods -o wide --all-namespaces
result:
NAMESPACE     NAME                                                 READY   STATUS    RESTARTS   AGE     IP               NODE                         NOMINATED NODE
kube-system   calico-node-22dbg                                    2/2     Running   0          11m     178.128.19.155   ubuntu-s-1vcpu-2gb-sgp1-01   <none>
kube-system   calico-node-tm46k                                    2/2     Running   0          5m12s   178.128.19.109   ubuntu-s-1vcpu-2gb-sgp1-02   <none>
kube-system   coredns-576cbf47c7-b2jvz                             1/1     Running   0          17m     192.168.0.2      ubuntu-s-1vcpu-2gb-sgp1-01   <none>
kube-system   coredns-576cbf47c7-v5k2c                             1/1     Running   0          17m     192.168.0.3      ubuntu-s-1vcpu-2gb-sgp1-01   <none>
kube-system   etcd-ubuntu-s-1vcpu-2gb-sgp1-01                      1/1     Running   0          16m     178.128.19.155   ubuntu-s-1vcpu-2gb-sgp1-01   <none>
kube-system   kube-apiserver-ubuntu-s-1vcpu-2gb-sgp1-01            1/1     Running   0          16m     178.128.19.155   ubuntu-s-1vcpu-2gb-sgp1-01   <none>
kube-system   kube-controller-manager-ubuntu-s-1vcpu-2gb-sgp1-01   1/1     Running   0          17m     178.128.19.155   ubuntu-s-1vcpu-2gb-sgp1-01   <none>
kube-system   kube-proxy-tncdf                                     1/1     Running   0          5m12s   178.128.19.109   ubuntu-s-1vcpu-2gb-sgp1-02   <none>
kube-system   kube-proxy-tq5dl                                     1/1     Running   0          17m     178.128.19.155   ubuntu-s-1vcpu-2gb-sgp1-01   <none>
kube-system   kube-scheduler-ubuntu-s-1vcpu-2gb-sgp1-01            1/1     Running   0          16m     178.128.19.155   ubuntu-s-1vcpu-2gb-sgp1-01   <none>
kube-system   **kubernetes-dashboard-77fd78f978-frk56**                1/1     Running   0          22s     192.168.1.2      ubuntu-s-1vcpu-2gb-sgp1-02   <none>
```

Enable proxy
(use screen or tmux to keep this )running
```
kubectl proxy
```
Establish a ssh tunnel to the server
```
[ local ]
ssh -fNT -L 8001:localhost:8001 test_m1

```
You should see the dashboard login page.
Next let's create the token to access the dashboard
```
[ Master ]
kubectl apply -f dashboard-user-binding.yml
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
result:
Data
====
token:      <token>
```
Go to dashboard and use token to sign-in
```
[ local brower ]
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default
```
For easy access to the dashboard you can use alias in your .bashrc or .zshrc
```
alias kubedash='ssh -fNT -L 8001:localhost:8001 test_m1; python -mwebbrowser "http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default"'

```

## Setting up NGINX ingress controller
### Reference
```
https://kubernetes.github.io/ingress-nginx/
```
Install Mandatory command
```
[ Master ]
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```

```
[ Master ]
kubectl apply -f baremetal-service-nodeport.yaml
```

## Deploying an API server to the cluster
```
kubectl apply -f deployment-api.yml
```
you should be able to access api (using Nodeport) by
```
curl http://ip:31001/
```
using your dns to point to the external ip you then can access the api by
```
curl mock.mydomain.com
```
