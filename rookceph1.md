## CephCluster

### 参考

https://rook.io/docs/rook/latest-release/CRDs/Cluster/ceph-cluster-crd/#cluster-settings

### 設定値メモ

- dataDirHostPath: クラスタを作り直すとき、dataDirHostPathのディレクトリ の削除が必要
- dashboard: enabledで有効化(ポートやパスも指定可)
- monitoring: enabledのときPrometheus有効化。デフォルトfalse
- network: ネットワークの設定
- mon: MONの設定
- mgr: MGRの設定
- crashCollector: crash collector daemonの設定
- logCollector: ログコレクターが動きログ出力(デフォルトtrue)
- placement: mon, mgrなどの配置を指定できる
- resources: リソースの上限を決められる
  - mon: 1024MB
  - mgr: 512MB
  - osd: 2048MB
  - crashcollector: 60MB
  - mgr-sidecar: 100MB limit, 40MB requests
  - prepareosd: no limits (see the note)
  - exporter: 128MB limit, 50MB requests
- storage: 使用するストレージの設定 ★
  - useAllNodes: 全ノードがクラスタ全体設定を使うかどうか true/false
  - useAllDevices: デバイス・パーティションが自動的に使われる　推奨されていない
  - deviceFilter: 正規表現で使用デバイス名を指定
  - devicePathFilter: 正規表現でデバイスパス名を指定
  - devices: nameでデバイス名を指定　パスでも良い
  - nodes: 個別のノードごと設定
- fullRatio: OSDがフルになったとしてIOをブロックする比率 95%
- backfillFullRatio: OSDがフルになったときバックフィル処理（データ再配置）を停止する閾値　 default is 0.90.
- nearFullRatio: フルになりそうな警告を上げる閾値 85%
- cleanupPolicy: クラスタ削除時のデータ消去方式
- security: 暗号化する場合のキーの運用について指定
- cephConfig: cephの設定

## CephBlockPool

### 設定値メモ

- replicated:
  - size: レプリカ数
  - requireSafeReplicaSize: デフォルトtrueだが、レプリカ数を1にするときには指定要
  - replicasPerFailureDomain: failureDomainごとのレプリカ数。hostで2なら、1 hostに2個、デフォルト1
- failuereDomain: osd or host
- deviceClass: プールが利用するssdなどのデバイスの種類を指定できる
- application: デフォルトrdb、rgwなどもある
- parameters: cephのパラメータの指定
  - compression_mode: 圧縮方式
- mirroring: rdb mirrorの設定
- quotas: クオータの設定
  - maxSize: プール全体の最大容量
  - maxObjects: 最大オブジェクト数

### 設定例 (cluster.yamlのstorage部分抜粋)

#### (1) disk by-pathを使う例

- cluster.yamlのstorage部分

```yaml
storage: # cluster level storage configuration and selection
  useAllNodes: false
  useAllDevices: false
  config:
    databaseSizeMB: "1024" # uncomment if the disks are smaller than 100 GB
  nodes:
    - name: "k8s-2"
      devices: # specific devices to use for storage can be specified for each node
        - name: "/dev/disk/by-path/virtio-pci-0000:08:00.0"
      config:
    - name: "k8s-3"
      devices: # specific devices to use for storage can be specified for each node
        - name: "/dev/disk/by-path/virtio-pci-0000:08:00.0"
      config:
    - name: "k8s-4"
      devices: # specific devices to use for storage can be specified for each node
        - name: "/dev/disk/by-path/virtio-pci-0000:08:00.0"
      config:
  allowDeviceClassUpdate: false # whether to allow changing the device class of an OSD after it is created
  allowOsdCrushWeightUpdate: false
```

#### (2) パーティションを使う例

- partision作成

```bash
sudo parted --script /dev/vdb mklabel gpt mkpart primary 1MiB 100MiB mkpart primary 100MiB 9900MiB
```

- cluster.yamlのstorage部分

```yaml
storage: # cluster level storage configuration and selection
  useAllNodes: false
  useAllDevices: false
  config:
    # crushRoot: "custom-root" # specify a non-default root label for the CRUSH map
    # metadataDevice: "md0" # specify a non-rotational storage so ceph-volume will use it as block db device of bluestore.
    databaseSizeMB: "1024" # uncomment if the disks are smaller than 100 GB
    # osdsPerDevice: "1" # this value can be overridden at the node or device level
    # encryptedDevice: "true" # the default value for this option is "false"
    # deviceClass: "myclass" # specify a device class for OSDs in the cluster
  nodes:
    - name: "k8s-2"
      devices: # specific devices to use for storage can be specified for each node
        - name: "vdb2"
      config:
    - name: "k8s-3"
      devices: # specific devices to use for storage can be specified for each node
        - name: "vdb2"
      config:
    - name: "k8s-4"
      devices: # specific devices to use for storage can be specified for each node
        - name: "vdb2"
      config:
  allowDeviceClassUpdate: false # whether to allow changing the device class of an OSD after it is created
  allowOsdCrushWeightUpdate: false # whether to allow resizing the OSD crush weight after osd pvc is increased
```

#### (3) LVMを使う例

- lvmの領域作成(/dev/mapper/pvvg-pvlv)

```bash
sudo pvcreate /dev/vdb
sudo vgcreate pvvg /dev/vdb
sudo lvcreate -l 100%FREE -n pvlv pvvg
```

- cluster.yamlのstorage部分

```yaml
storage: # cluster level storage configuration and selection
  useAllNodes: false
  useAllDevices: false
  config:
    # crushRoot: "custom-root" # specify a non-default root label for the CRUSH map
    # metadataDevice: "md0" # specify a non-rotational storage so ceph-volume will use it as block db device of bluestore.
    databaseSizeMB: "1024" # uncomment if the disks are smaller than 100 GB
    # osdsPerDevice: "1" # this value can be overridden at the node or device level
    # encryptedDevice: "true" # the default value for this option is "false"
    # deviceClass: "myclass" # specify a device class for OSDs in the cluster
  nodes:
    - name: "k8s-2"
      devices: # specific devices to use for storage can be specified for each node
        - name: "/dev/mapper/pvvg-pvlv"
      config:
    - name: "k8s-3"
      devices: # specific devices to use for storage can be specified for each node
        - name: "/dev/mapper/pvvg-pvlv"
      config:
    - name: "k8s-4"
      devices: # specific devices to use for storage can be specified for each node
        - name: "/dev/mapper/pvvg-pvlv"
      config:
  allowDeviceClassUpdate: false # whether to allow changing the device class of an OSD after it is created
  allowOsdCrushWeightUpdate: false # whether to allow resizing the OSD crush weight after osd pvc is increased
```

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com # csi-provisioner-name
parameters:
  clusterID: rook-ceph # namespace:cluster
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/fstype: ext4
allowVolumeExpansion: true
reclaimPolicy: Delete
```

## dashborad (NodePort)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-mgr-dashboard-external-https
  namespace: rook-ceph
  labels:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
spec:
  ports:
    - name: dashboard
      port: 8443
      protocol: TCP
      targetPort: 8443
      nodePort: 30443
  selector:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
    mgr_role: active
  sessionAffinity: None
  type: NodePort
```

## 初期構築

### 参考

- https://rook.io/docs/rook/latest-release/Getting-Started/quickstart/
- https://rook.io/docs/rook/latest-release/Storage-Configuration/- Block-Storage-RBD/block-storage/#provision-storage

### 手順

1. CRD, RBAC等作成

   以下を実行して、operatorのPodがRunningになるまで待つ。  
   この段階で、必要なCRDなどが作成される。

   ```bash
   git clone --single-branch --branch v1.17.7 https://github.com/rook/rook.git
   cd rook/deploy/examples
   kubectl create -f crds.yaml -f common.yaml -f operator.yaml
   kubectl -n rook-ceph get pod
   ```

2. CephCluster作成

   利用するディスク設定などをcluster.yamlに記述して、以下を実行する。  
   実行するとCephClusterが作成される。  
   5分程度待つと、問題なければOSDのPodが作成され、cephクラスターの構築が完了する。

   ```bash
   kubectl create -f cluster.yaml
   ```

3. ツールボックスで状態確認

   ```bash
   kubectl create -f toolbox.yaml
   kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
   ceph status
   ceph df
   ceph osd df tree
   ceph osd tree
   ceph report　　→利用ディスク確認可能
   ceph osd metadata →利用ディスク確認可能
   ```

4. blockストレージ利用の設定

   以下を実行することにより、blockストレージ用のstorageclassが作成される。

   ```bash
   cd $HOME/rook
   cd deploy/examples/csi/rbd/
   kubectl create -f storageclass.yaml
   kubectl patch storageclass rook-ceph-block -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
   ```

5. dashboard利用の設定

   上記のdashboardのyamlからNodePort用のサービスを作成する。

   ```
   kubectl create -f dashboard.yaml
   ```

   パスワードの確認方法は以下。

   ```bash
   kubectl get secret rook-ceph-dashboard-password -o yaml -n rook-ceph
   ```
