### Prometheus-operator监控外部etcd



> 目前etcd是二进制安装的，所以Prometheus-operator监控etcd就相当于监控外部部署的etcd。采坑记录如下。

- 按照之前的方法创建svc和endpoint，这个地方正常创建：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd-k8s
  labels:
    k8s-app: etcd
    namespace: kube-system
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: api
    port: 2379
    protocol: TCP
---
apiVersion: v1
kind: Endpoints
metadata:
  name: etcd-k8s
  labels:
    k8s-app: etcd
    namespace: kube-system
subsets:
- addresses:
  - ip: 192.168.150.29
    nodeName: k8s11-1
  ports:
  - name: api
    port: 2379
    protocol: TCP
```

> Tips: etcd的监控地址在https://192.168.150.29:2379/metrics

- 创建etcd的servicemonitor

```yaml
# cat etcd-servicemonitor.yaml 
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd-k8s
  namespace: monitoring
  labels:
    k8s-app: etcd
    prometheus: k8s
spec:
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: etcd
  endpoints:
  - interval: 10s
    port: api
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-certs/ca.pem
      certFile: /etc/prometheus/secrets/etcd-certs/etcd.pem
      keyFile: /etc/prometheus/secrets/etcd-certs/etcd-key.pem
      #use insecureSkipVerify only if you cannot use a Subject Alternative Name
      insecure_skip_verify: true
      #serverName: 192.168.150.29
```

​	创建该servicemonitor最大的问题在于证书的问题，一开始我创建的时候是指定的本地安装etcd使用的证书地址，然后就一直报错招不到证书：

```shell
level=error ts=2018-12-13T02:53:55.540345792Z caller=scrape.go:240 component="scrape manager" scrape_pool=monitoring/etcd-k8s/0 msg="Error creating HTTP client" err="unable to use specified CA cert /etc/etcd/ssl/ca.pem: open /etc/etcd/ssl/ca.pem: no such file or directory"
```

​	后来查找资料发现，Prometheus是在自己的pod内部查找该证书的，所以写的宿主机etcd的地址跟他没有关系，然后就创建一个secrets ：

```shell
# kubectl -n monitoring create secret generic etcd-certs --from-file=/etc/etcd/ssl/ca.pem --from-file=/etc/etcd/ssl/etcd.pem --from-file=/etc/etcd/ssl/etcd-key.pem 
secret/etcd-certs created
```

​	创建完证书之后，需要Prometheus挂载进去，所以就要修改创建Prometheus的文件`vim prometheus-prometheus.yaml`,添加以下证书相关内容，然后apply使其生效：

```yaml
# cat prometheus-prometheus.yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    prometheus: k8s
  name: k8s
  namespace: monitoring
spec:
  alerting:
    alertmanagers:
    - name: alertmanager-main
      namespace: monitoring
      port: web
  baseImage: quay.io/prometheus/prometheus
  nodeSelector:
    beta.kubernetes.io/os: linux
  replicas: 2
  resources:
    requests:
      memory: 400Mi
  ruleSelector:
    matchLabels:
      prometheus: k8s
      role: alert-rules
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  secrets:  # 添加该内容
  - etcd-certs
  version: v2.5.0
```

​	这个时候我们进入Prometheus的pod内部查看该证书的路径：

```shell
/ $ df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                  40.0G     14.6G     25.4G  36% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                     3.9G         0      3.9G   0% /sys/fs/cgroup
/dev/sda1                40.0G     14.6G     25.4G  36% /etc/hostname
/dev/sda1                40.0G     14.6G     25.4G  36% /etc/prometheus/config_out
tmpfs                     3.9G     12.0K      3.9G   0% /etc/prometheus/secrets/etcd-certs

```

​	路径就在`/etc/prometheus/secrets/etcd-certs`下面，所以上面创建的servicemonitor就使用该路径。

![1544673513213](D:\学习文档\markdown\%5CUsers%5Cpenn%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5C1544673513213.png)



参考：[monitor external etcd](https://github.com/coreos/prometheus-operator/blob/v0.19.0/contrib/kube-prometheus/docs/Monitoring%20external%20etcd.md)

