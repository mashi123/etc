# kvm操作メモ
## 仮想マシン操作
### 仮想マシン一覧
```bash
virsh list --all
```

### 起動
```bash
virsh start ubuntu2404
```

### 接続
```bash
virsh console ubuntu2404
```

### 接続解除
`Ctrl+]`
`Ctrl+5`

### 停止
```bash
virsh shutdown ubuntu2404
```

### 強制停止
```bash
virsh destroy ubuntu2404
```

### クローン
```bash
virt-clone --original ubuntu2404-server --name k8s-1 --auto-clone 
```

## ネットワーク関連
### ネットワーク(表示)
```bash
virsh net-list
virsh net-dumpxml default
```

### プライベートネットワーク作成
- virbr2.xmlを以下の内容で作成。
```xml
<network>
  <name>private</name>
  <bridge name="virbr2"/>
  <ip address="192.168.200.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.200.2" end="192.168.200.254"/>
    </dhcp>
  </ip>
</network>
```

- ネットワーク作成および活性化
```bash
virsh net-define virbr2.xml
virsh net-list --all
virsh net-start private
virsh net-autostart private
```

#### 参考
- https://libvirt.org/formatnetwork.html#isolated-network-config
- https://unix.stackexchange.com/questions/721794/how-to-create-internal-network-using-libvirt-qemu-kvm-stack

### ディスクイメージ作成(qcow2)
```bash
qemu-img create -f qcow2 /var/lib/libvirt/images/new_disk.qcow2 10G
```

### ディスク情報
```bash
sudo qemu-img info /var/lib/libvirt/images/ubuntu2404.img
```

### ディスク追加(/dev/sdbの場合)
```bash
virsh attach-disk ubuntu2404 --source /var/lib/libvirt/images/new_disk.qcow2 --target vdb --driver qemu  --subdriver qcow2 --type disk --persistent
```

### ディスク取り外し
```bash
virsh detach-disk ubuntu2404 vdb --persistent
```

### インストール
```bash
virt-install \
--name ubuntu2404-server \
--ram 2048 \
--disk path=/var/lib/libvirt/images/ubuntu2404-server.img,size=20 \
--vcpus 2 \
--os-variant  ubuntu24.04 \
--network network=default \
--network network=private \
--graphics none \
--console pty,target_type=serial \
--location /home/ubuntu-24.04.1-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd \
--extra-args 'console=ttyS0,115200n8'
```
