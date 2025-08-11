## 事前準備

ノードのroleがworkerを表すlabelをつける。

```bash
kubectl label node k8s-2 node-role.kubernetes.io/worker=
kubectl label node k8s-3 node-role.kubernetes.io/worker=
kubectl label node k8s-4 node-role.kubernetes.io/worker=
```

## 停止

1. スケジュール抑止

```bash
for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do echo ${node} ; kubectl cordon ${node} ; done
```

1. pod退避(drain)

```bash
for node in $(kubectl get nodes -l node-role.kubernetes.io/worker -o jsonpath='{.items[*].metadata.name}'); do echo ${node} ; kubectl drain ${node} --delete-emptydir-data --ignore-daemonsets=true --timeout=15s --force ; done
```

1. pod退避(PDBを無効にしてdrain再実行)

```bash
for node in $(kubectl get nodes -l node-role.kubernetes.io/worker -o jsonpath='{.items[*].metadata.name}'); do echo ${node} ; kubectl drain ${node} --delete-emptydir-data --ignore-daemonsets=true --disable-eviction --timeout=15s --force ; done
```

1. 再起動後

```bash
for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do echo ${node} ; kubectl uncordon ${node} ; done
```
