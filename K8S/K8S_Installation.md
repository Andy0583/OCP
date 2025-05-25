### 前置作業
---
> 準備三台Ubuntu VM
```
sudo ufw disable
sudo passwd root
sudo vi /etc/ssh/sshd_config
    PermitRootLogin yes
sudo systemctl restart ssh
```

> 可使用Putty登入root帳號
```
apt update -y && apt upgrade -y
```

> 依據環境不同修改Host Name及”hosts”
```
hostnamectl set-hostname "k8s1.andy.com"
cat >> /etc/hosts << EOF
172.12.25.51 k8s1.andy.com
172.12.25.52 k8s2.andy.com
172.12.25.53 k8s3.andy.com
EOF
```

> 永久關閉Swap
```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 安裝K8S
---
```
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```
```
apt-get -y install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    apt-transport-https \
    gpg

mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null  
apt-get update
```

> 安裝Containerd.io 容器化運行平台
```
apt-get install containerd.io  
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd
```

> 請依據所需K8S版本號碼輸入，並進行安裝kubelet、kubeadm、kubectl。
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-cache madison kubelet
apt install -y kubelet kubeadm  kubectl
apt-mark hold kubelet kubeadm kubectl
modprobe br_netfilter
```

### K8S初始化
> K8S初始化僅針對Master執行，control-plane-endpoint請輸入Master IP
```
kubeadm init --control-plane-endpoint="172.12.25.51" --pod-network-cidr=10.244.0.0/16
```
> 紀錄下圖"Then you can join any number of worker nodes by running the following on each as root"資訊
  
![](https://github.com/Andy0583/OCP/blob/main/Image/k8s/k8s-1.png?raw=true)

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml
```

> 若需重新顯示Join Master資訊
```
kubeadm token create --print-join-command
```

> Worker Node加入K8S Cluster
> 於Worker執行。加入Cluster，使用Master產出的Join資訊
```
kubeadm join 192.168.0.231:6443 --token 0bxz18.pl91tl6wuovyi04i \
        --discovery-token-ca-cert-hash sha256:860dfa06a3f7614fa4fa404fa8647ae06ce001b52f2a469bc2c76d93cc59d174
```

### 簡化指令（非必要）
--
```
cat >> /etc/profile << EOF
alias kg="kubectl get"
alias kc="kubectl create"
alias kcf="kubectl create -f"
alias kdf="kubectl delete -f"
alias kds="kubectl describe”
EOF
source /etc/profile
```

### 確認安裝成功
--
* 於Master執行，於Master檢查Node資訊，確認安裝成功
```
root@k8s1:~# kubectl get node
NAME            STATUS   ROLES           AGE     VERSION
k8s1.andy.com   Ready    control-plane   14m     v1.27.10
k8s2.andy.com   Ready    <none>          3m16s   v1.27.10
k8s3.andy.com   Ready    <none>          3m13s   v1.27.10
```

> 測試K8S是否可用
```
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort

kubectl get svc nginx
NAME    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.99.22.145   <none>        80:32400/TCP   2m11s
```

> 開啟瀏覽器輸入 "K8S IP : Port"，若成功會顯示如下圖 </p>

![](https://github.com/Andy0583/OCP/blob/main/Image/k8s/k8s-2.png?raw=true)

> 刪除測試資料
```
root@k8s1:~# kubectl delete svc nginx
service "nginx" deleted

root@k8s1:~# kubectl delete deploy nginx
deployment.apps "nginx" deleted
```
