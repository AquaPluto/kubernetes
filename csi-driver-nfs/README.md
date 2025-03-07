# 部署CSI-NFS-Deriver

必要的步骤说明：

1. 需要事先有可用的NFS服务，且能够满足CSI-NFS-Driver的使用要求；
2. 在Kubernetes集群上部署NFS CSI Driver；
3. 在Kubernetes集群上创建StorageClass资源，其provisioner为nfs.csi.k8s.io，而parameters.server要指向准备好的NFS Server的访问入口；
4. 测试使用；

## 部署NFS Server
- 运行如下命令，在Kubernetes集群上部署一个测试可用的NFS服务；

```bash
kubectl create namespace nfs
wget https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/deploy/example/nfs-provisioner/nfs-server.yaml
sed -i 's/namespace: default/namespace: nfs/g' nfs-server.yaml
kubectl apply -f nfs-server.yaml --namespace nfs
```

- 也可以自行部署NFS Server，并按需export相应的目录；以“/data”目录为例，其export的示例如下；

```
/data *(rw,fsid=0,async,no_subtree_check,no_auth_nlm,insecure,no_root_squash)
```

## 部署 NFS CSI Driver

支持两种部署方式：远程部署和本地部署，前者是指直接从远程仓库中获取部署配置文件完成的部署，而后者则需要首先克隆csi-driver-nfs项目的仓库至本地，并在克隆而来的本地项目目录中进行部署。

目前，NFS CSI Driver项目维护有多个不同的版本，部署前需要首先确定要选择使用的版本。

> 需要特别说明的是，v4.0、v4.1及v4.2几个版本中，在csi-nfs-controller和csi-nfs-node相关的Pod的配置中的dnsPolicy都使用了“Default”，但又使用了spec.hostNetwork配置，这种配置中，此两者相关的Pod将无法使用ClusterDNS解析集群上的服务。因此，为了能够同此前的部署NFS Server协同，我们需要事先修改dnsPolicy的值为“ClusterFirstWithHostNet”。v4.3及其之后的版本中，csi-nfs-controller和csi-nfs-node相关的Pod的配置中的dnsPolicy已经设定使用“ClusterFirstWithHostNet”，因而无须再改。

以v4.6.0为例，相关的文件在[deploy/csi-driver-nfs-4.6.0](https://github.com/AquaPluto/kubernetes/tree/main/csi-driver-nfs/deploy/csi-driver-nfs-4.6.0)目录下。本示例将直接基于这些文件完成NFS CSI Driver的部署。

 - local install（基于当前项目的部署，其默认配置已经修改dnsPolicy）
```console
git clone https://github.com/AquaPluto/kubernetes.git
cd kubernetes
kubectl apply -f csi-driver-nfs/deploy/csi-driver-nfs-4.6.0/
```

- check pods status:
```console
kubectl -n kube-system get pod -o wide -l 'app in (csi-nfs-node,csi-nfs-controller)'
```

example output:

```console
NAME                                  READY   STATUS    RESTARTS        AGE     IP           NODE             NOMINATED NODE   READINESS GATES
csi-nfs-controller-76697d88bc-25rw7   4/4     Running   2 (3m10s ago)   5m59s   10.0.0.184   node1.wu.org     <none>           <none>
csi-nfs-node-4nt8c                    3/3     Running   1 (4m45s ago)   5m59s   10.0.0.183   master1.wu.org   <none>           <none>
csi-nfs-node-k4k5w                    3/3     Running   1 (3m54s ago)   5m59s   10.0.0.186   node2.wu.org     <none>           <none>
csi-nfs-node-s4q2z                    3/3     Running   1 (3m30s ago)   5m59s   10.0.0.184   node1.wu.org     <none>           <none>
```


## 创建StorageClass

 -  Create a StorageClass
 > 将 `server`、`share` 更改为你现有的 NFS 服务器地址和共享名称
```
kubectl create -f https://raw.githubusercontent.com/AquaPluto/kubernetes/refs/heads/main/csi-driver-nfs/nfs-csi-storageclass.yaml
```

若需要在创建完StorageClass后将其设置为默认，可使用类似如下命令进行。

```bash
kubectl patch storageclass nfs-csi -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

取消某StorageClass的默认设定，则将类似上面命令中的annotation的值修改为false即可。

## 创建PVC进行测试

```console
kubectl create -f https://raw.githubusercontent.com/AquaPluto/kubernetes/refs/heads/main/csi-driver-nfs/nfs-pvc-dynamic.yaml
```
example output:
```
~#kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   REASON   AGE
pvc-a9a84173-7308-46ac-99e8-c140cc7f012e   10Gi       RWX            Delete           Bound    default/pvc-nfs-dynamic   nfs-csi                 7s

~#kubectl get pvc
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-nfs-dynamic   Bound    pvc-a9a84173-7308-46ac-99e8-c140cc7f012e   10Gi       RWX            nfs-csi        18s
```
## 创建Pod进行测试
```console
kubectl create -f https://raw.githubusercontent.com/AquaPluto/kubernetes/refs/heads/main/csi-driver-nfs/volumes-nfs-demo.yaml
```