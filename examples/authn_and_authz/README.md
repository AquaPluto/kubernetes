# 静态令牌认证
第一步：生成token命令
```bash
echo "$(openssl rand -hex 3).$(openssl rand -hex 8)"
```

第二步：生成static token文件，文件内容示例
```bash
b986e0.3ec5c9d82b5e7503,tom,1001,kubeadmin
6fd20a.fde3a95eb81d12c4,jerry,1002,kubeuser
```

第三步：配置kube-apiserver加载该静态令牌文件以启用相应的认证功能（注意不要在原文件中直接改）
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --token-auth-file=/etc/kubernetes/authfiles/token.csv
    volumeMounts:
    - mountPath: /etc/kubernetes/authfiles/token.csv
      name: static-tokens
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/authfiles/token.csv
      type: FileOrCreate
    name: static-tokens
```

第四步：测试
```bash
#使用kubectl命令
kubectl --server=https://API_SERVER:6443 --token="b986e0.3ec5c9d82b5e7503" --certificate-authority=/etc/kubernetes/pki/ca.crt get pods

#使用curl命令
curl -k -H "Authorization: Bearer b986e0.3ec5c9d82b5e7503" https://API_SERVER:6443/api/v1/namespaces/default/pods/
```
# X509数字证书认证
第一步：生成私钥
```bash
cd /etc/kubernetes/pki/
(umask 077; openssl genrsa -out mason.key 4096)
```

第二步：根据私钥创建证书签署请求文件
```bash
openssl req -new -key ./mason.key -out ./mason.csr -subj "/CN=mason/O=developers"
```

第三步：由Kubernetes CA签署证书
```bash
openssl x509 -req -days 365 -CA ./ca.crt -CAkey ./ca.key -CAcreateserial -in ./mason.csr -out ./mason.crt
```

第四步：将pki目录下的 mason.crt、mason.key 和 ca.crt 复制到某部署了kubectl的主机上，即可进行测试
```bash
scp -rp ./{mason.crt,mason.key} k8s-node01:/etc/kubernetes/pki

#使用kubectl测试
kubectl get pods --client-certificate=$HOME/.certs/mason.crt --client-key=$HOME/.certs/mason.key --server=https://API_SERVER:6443/ --certificate-authority=/etc/kubernetes/pki/ca.crt

#使用curl命令
curl --cert ./mason.crt --key ./mason.key --cacert ./ca.crt  https://API_SERVER:6443/api/v1/namespaces/default/pods 
```