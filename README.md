# Deploy Simple App to Kubernetes Cluster using Jenkins

### Eksekusi disemua node

#### Set name resolution

sudo vim /etc/hosts
```
10.0.1.97 an-master0
10.0.1.100 an-worker0
10.0.1.101 an-worker1
```
#### Update & Install package
```
apt update && apt upgrade -y
sudo apt install -y docker.io; sudo docker version
nano /etc/docker/daemon.json
...
{
   "insecure-registries" : ["10.0.1.97:5000"]
}
...
nano /lib/systemd/system/docker.service
#edit line like this :
...
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H fd:// --containerd=/run/containerd/containerd.sock
...
systemctl daemon-reload
sudo service docker restart
```
#### test Docker remote access
```
docker -H tcp://10.0.1.97:2375 version
docker -H tcp://10.0.1.101:2375 version
docker -H tcp://10.0.1.100:2375 version
```

#### Eksekusi di node an-master0

#### Create Private Docker registry
 ```
sudo docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

### Eksekusi disemua node

#### Install k8s
```
sudo apt install -y apt-transport-https; curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo vi /etc/apt/sources.list.d/kubernetes.list
...
deb http://apt.kubernetes.io/ kubernetes-xenial main
...
sudo apt update; sudo apt install -y kubectl=1.15.4-00 kubelet=1.15.4-00 kubeadm=1.15.4-00
```

### Eksekusi di node an-master0
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
#### Instal POD Network Flannel
```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml
kubectl get pods --all-namespaces --watch
```
#### Tampilkan Token dan token-ca-cert-hash
```
sudo kubeadm token list
sudo openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```
### Eksekusi di node an-worker0 dan an-worker1

* Join Node Worker ke Master (pastikan swap tidak menyala)
```
swapon -s
sudo swapoff -a
sudo kubeadm join --token [TOKEN] [NODE-MASTER]:6443 --discovery-token-ca-cert-hash sha256:[TOKEN-CA-CERT-HASH]
```

### Eksekusi di node an-master0

#### Clone repository
```
git clone https://github.com/misskecupbung/jenkins-k8s.git
cd jenkins-k8s/modified-images/
```
#### build images
```
docker build -t misskecupbung/jenkins-k8s:latest .
docker login
docker push misskecupbung/jenkins-k8s:latest
```
#### create deployment and service
```
kubectl apply -f jenkins-deployment.yaml 
kubectl apply -f jenkins-service.yaml 
kubectl get deployment,pod,service
```
#### Login ke Jenkins Dashboard
```
http://an-master0:30100
```

#### install plugin
Login -> Manage Jenkins -> Manage Plugins -> Available :
   * Blue Ocean
   * Kubernetes :: Pipeline :: DevOps Steps	
   * Kubernetes Cli Plugin
   * Kubernetes Client API Plugin	
   * Kubernetes Continuous Deploy Plugin
   * Kubernetes Credentials Plugin
   * Kubernetes plugin
   
#### Manage Kubernetes

Login -> Manage Jenkins -> Configure System -> Add a new cloud > Kubernetes :
   * Kubernetes URL : http://10.60.8.133:30100/
   * Add Container Template :
       * Namespace : jenkins
       * Container name : jnlp
       * Docker images : joao29a/jnlp-slave-alpine-docker
   * Apply/Save

#### Create credential

Login -> Credentials -> Jenkins -> Global credentials (unrestricted) -> Add Credential :
   * kind: Kubernetes Configuration (kubeconfig)
   * id: kubeconfig
   * enter directy. (paste from `/root/.kube/config` in master node.)

#### Deploy App

Login -> Open Blue Ocean -> New Pipeline -> GitHub -> Select repository (Ex: jenkins-k8s) -> Create Pipeline

#### Test app

kubectl get deployment
kubectl get pods
kubectl get services
curl an-master0:32313

#### Set Time Trigger deploy

Login -> Jenkins-k8s -> Configure -> Scan repository triggers -> Periodically if not otherwise run -> Set interval: 1 minutes.
