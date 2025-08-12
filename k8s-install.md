## オンプレ環境にk8sをインストール

- 対象OSはUbuntu 24.04
- コンテナエンジンにdockerを利用

## 事前定義

```bash
NODE_ADDR=192.168.200.101
```

### dockerインストール

https://docs.docker.com/engine/install/ubuntu/

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### cri-dockerd設定

```bash
curl -LO https://github.com/Mirantis/cri-dockerd/releases/download/v0.4.0/cri-dockerd-0.4.0.amd64.tgz
tar xvfz cri-dockerd-0.4.0.amd64.tgz
sudo cp cri-dockerd/cri-dockerd /usr/bin
sudo chmod 655 /usr/bin/cri-dockerd

sudo curl -L https://raw.githubusercontent.com/Mirantis/cri-dockerd/refs/heads/master/packaging/systemd/cri-docker.service -o /etc/systemd/system/cri-docker.service
sudo curl -L https://raw.githubusercontent.com/Mirantis/cri-dockerd/refs/heads/master/packaging/systemd/cri-docker.socket -o /etc/systemd/system/cri-docker.socket
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service --now
```

### swap無効化

```bash
sudo sed  -i -e "s,^/swap.img\(.*$\),#/swap.img\1," /etc/fstab
sudo swapoff -a
```

### kubeadmインストール

```bash
sudo apt-get update
### apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

### If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
### sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

### This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### node ipの設定

```bash
sudo sed -i -e "s/KUBELET_EXTRA_ARGS=/KUBELET_EXTRA_ARGS=--node-ip=${NODE_ADDR}/" /etc/default/kubelet

sudo systemctl enable --now kubelet
```

### kubeadm init (Master only)

```bash
sudo kubeadm init --apiserver-advertise-address 192.168.200.101 --pod-network-cidr 10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Calico (Master only)

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/custom-resources.yaml -O
sed -i -e "s,^      cidr: 192.168.0.0/16,      cidr: 10.244.0.0/16," custom-resources.yaml
kubectl create -f custom-resources.yaml
```

### worker node追加 (Worker only (token, ca_hashは値コピー))

```bash
token=d1bbzw.u1uc7egeyuw4r6nx
ca_hash=2ab21c9e0778dbdc1ad34367b05e51b61187724462b57c552c55413093187aca

sudo kubeadm join 192.168.200.101:6443 --token ${token} --discovery-token-ca-cert-hash sha256:${ca_hash} --cri-socket unix:///var/run/cri-dockerd.sock
```
