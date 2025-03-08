# MetalLB简单应用

使用标准路由协议的 Kubernetes 网络负载均衡器实现。

MetalLB核心功能的实现依赖于两种机制：

- 地址分配：基于指定的地址池进行分配；
- 对外公告：让集群外部的网络了解新分配的IP地址，MetalLB使用ARP、NDP或BGP实现

### 部署MetalLB

kube-proxy工作于ipvs模式时，必须要使用严格ARP（StrictARP）模式，因此，若有必要，先运行如下命令，配置kube-proxy。

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

随后，运行如下命令，即可部署MetalLB至Kubernetes集群。

```bash
METALLB_VERSION='v0.14.9'
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/${METALLB_VERSION}/config/manifests/metallb-native.yaml
```

### 创建Layer2模式地址池

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: localip-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.51-10.0.0.80
  autoAssign: true
  avoidBuggyIPs: true
```

### 创建二层公告机制

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: localip-pool-l2a
  namespace: metallb-system
spec:
  ipAddressPools:
  - localip-pool
  interfaces:
  - eth0
```
