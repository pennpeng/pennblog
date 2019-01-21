### Prometheus-operator监控其他namespace数据

> Prometheus operator默认只是获取default、kube-system、monitoring这三个namespace的数据，我创建一个rook集群，并创建一个ceph的servicemonitor，在Prometheus的configuration里面已经生成了job配置信息，但是在target和discovery里面并没有该信息。

- Prometheus日志
  查看日志如下报错：
```shell
level=error ts=2018-12-12T06:49:35.57698563Z caller=main.go:240 component=k8s_client_runtime err="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:302: Failed to list *v1.Pod: pods is forbidden: User \"system:serviceaccount:monitoring:prometheus-k8s\" cannot list pods in the namespace \"rook-ceph\""
level=error ts=2018-12-12T06:49:35.577001264Z caller=main.go:240 component=k8s_client_runtime err="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:301: Failed to list *v1.Service: services is forbidden: User \"system:serviceaccount:monitoring:prometheus-k8s\" cannot list services in the namespace \"rook-ceph\""
level=error ts=2018-12-12T06:49:35.577026562Z caller=main.go:240 component=k8s_client_runtime err="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:300: Failed to list *v1.Endpoints: endpoints is forbidden: User \"system:serviceaccount:monitoring:prometheus-k8s\" cannot list endpoints in the namespace \"rook-ceph\""
```
根据日志判断是namespace权限问题，查询安装的role、clusterrole相关文件，发现有两个文件是跟namespace相关，分别是`prometheus-roleBindingSpecificNamespaces.yaml`和` prometheus-roleSpecificNamespaces.yaml`。
- 解决

所以修改以上两个文件，添加需要监控的namespace相关权限，追加以下内容：
```yaml
# vim prometheus-roleSpecificNamespaces.yaml
- apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: prometheus-k8s
    namespace: rook-ceph
  rules:
  - apiGroups:
    - ""
    resources:
    - nodes
    - services
    - endpoints
    - pods
    verbs:
    - get
    - list
    - watch
    
# vim prometheus-roleBindingSpecificNamespaces.yaml
- apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: prometheus-k8s
    namespace: rook-ceph
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: prometheus-k8s
  subjects:
  - kind: ServiceAccount
    name: prometheus-k8s
    namespace: monitoring #注意这个namespace不要写错了，一定要为Prometheus部署的ns monitoring

```
这样再创建servicemonitor就可以了，而且这个servicemonitor的namespace也可以为rook-ceph，因为Prometheus已经有权限了。

:1st_place_medal:

:1234:

