## nginx ingress配置ssl passthrough

> 安装的rook ceph dashboard的pod本身是https访问的，所以ingress访问的时候有问题，需要将tls passthrough。

#### 第一步要配置nginx ingress支持该功能

默认是不开启支持的，需要在安装的时候开启该参数支持.

> !!! note SSL Passthrough is **disabled by default** and requires starting the controller with the [`--enable-ssl-passthrough`](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/cli-arguments.md) flag.

#### 第二步创建ingress如下：

```yaml
# cat ceph-dashboard.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-ceph
  namespace: rook-ceph
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    #ingress.kubernetes.io/secure-backends: "true"
spec:
  tls:
  - hosts:
    - ceph.172.50.0.29.nip.io
  rules:
  - host: ceph.172.50.0.29.nip.io
    http:
      paths:
      - backend:
          serviceName: rook-ceph-mgr-dashboard
          servicePort: 8443
```

主要是添加`nginx.ingress.kubernetes.io/ssl-passthrough: "true"`，在添加该annotations的时候一开始根据网上的教程添加的是`ingress.kubernetes.io/ssl-passthrough: "true"`，始终无法访问。最后对比官方参数发现少了nginx。

#### 参考：[nginx annotation](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md#ssl-passthrough)

[Advanced Ingress Configuration](https://docs.giantswarm.io/guides/advanced-ingress-configuration/)



