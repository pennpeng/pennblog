## Prometheus operator监控

> 在部署完Prometheus operator之后发现有几个target没有获取到数据，主要有三个：coredns、kube-controller-manager、kube-scheduler；接下来逐步分析并获取到数据

### coredns数据

查看coredns的servicemonitor是如何定义的：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: coredns
  name: coredns
  namespace: monitoring
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    interval: 15s
    port: metrics
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: kube-dns
```

上面的servicemonitor定义是从kube-system的namespace获取`k8s-app: kube-dns`这个label的svc信息，并且从metrics端口获取数据。

查看现在环境coredns的svc信息并没有相关metrics的端口信息：

```yaml
[root@k8s11-1 ~]# kubectl describe svc kube-dns --namespace=kube-system 
Name:              kube-dns
Namespace:         kube-system
Labels:            addonmanager.kubernetes.io/mode=Reconcile
                   k8s-app=kube-dns
                   kubernetes.io/cluster-service=true
                   kubernetes.io/name=CoreDNS
Annotations:       kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"prometheus.io/scrape":"true"},"labels":{"addonmanager.kubernetes.io/mode":"Reconcile","...
                   prometheus.io/scrape=true
Selector:          k8s-app=kube-dns
Type:              ClusterIP
IP:                10.68.0.2
Port:              dns  53/UDP
TargetPort:        53/UDP
Endpoints:         172.20.0.157:53,172.20.2.200:53
Port:              dns-tcp  53/TCP
TargetPort:        53/TCP
Endpoints:         172.20.0.157:53,172.20.2.200:53
Session Affinity:  None
Events:            <none>

```

coredns的Prometheus数据获取端口在9153，且coredns的版本要大于1.2.2否则有bug。所以解决办法就是删除之前的svc，创建一个新的svc并指定metrics端口如下：

```yaml
[root@k8s11-1 ~]# cat coredns-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.68.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
    targetPort: 9153
```

有了metrics端口之后，Prometheus就能获取到数据了

![1544673805709](https://github.com/pennpeng/pennblog/blob/master/images/2019-01-21_102944.png?raw=true)



-  参考：[coredns数据被Prometheus抓取](https://github.com/gjmzj/kubeasz/issues/322)
---
### kube-controller数据

查看下kube-controller的servicemonitor是如何定义的：
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    metricRelabelings:
    - action: drop
      regex: etcd_(debugging|disk|request|server).*
      sourceLabels:
      - __name__
    port: http-metrics
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: kube-controller-manager
```
>从kube-system的命名空间寻找label为k8s-app: kube-controller-manager的svc，且端口为http-metrics。
>实际上我是使用二进制部署的集群，没有相应的svc，所以需要手动创建该svc以及对应的endpoint。

查询发现系统已经创建了一个默认的endpoint，且无法删除，因为删除之后系统会自动创建：
```
[root@k8s11-1 manifests]# kubectl get endpoints 
NAME                      ENDPOINTS                                                                   AGE
fuseim.pri-ifs            <none>                                                                      112d
kube-controller-manager   <none>                                                                      26m
kube-dns                  172.20.0.157:53,172.20.2.200:53,172.20.0.157:53 + 3 more...                 5h
kube-scheduler            <none>                                                                      112d
kubelet                   192.168.150.10:10255,192.168.150.29:10255,192.168.150.6:10255 + 6 more...   112d
```
所以需要创建一个其他名字的svc和endpoint，在创建过程中趟了一个坑，就是创建该svc不能有selector和clusterip，因为如果有selector则会自动创建endpoint没法获取到系统信息；如果有clusterip则不能跟我们的endpoint匹配上，我们创建的endpoint的ip地址是指定系统ip的，因为是二进制安装的，所以获取的是宿主机ip。

在创建svc的时候还要匹配上面的servicemonitor相关label信息，最终创建如下：
```yaml
# cat kube-controller-endpoints.yaml 
kind: Endpoints
apiVersion: v1
metadata:
  namespace: kube-system
  name: kube-controller
  labels:
    k8s-app: kube-controller-manager
subsets:
  - addresses:
    - ip: 192.168.150.29
    ports:
    - name: http-metrics
      port: 10252
      protocol: TCP
      
# cat kube-controller-endpoints.yaml 
kind: Endpoints
apiVersion: v1
metadata:
  namespace: kube-system
  name: kube-controller
  labels:
    k8s-app: kube-controller-manager
subsets:
  - addresses:
    - ip: 192.168.150.29
    ports:
    - name: http-metrics
      port: 10252
      protocol: TCP
```
刷新获取kube-controller信息

---
### kube-scheduler数据
步骤如上，测试发现系统虽然创建了endpoint，但是我们可以apply我们创建的，让他生效，从而使用系统的endpoint。
```yaml

# cat kube-scheduler-svc.yaml 
kind: Service
apiVersion: v1
metadata:
  namespace: kube-system
  name: kube-scheduler
  labels:
    k8s-app: kube-scheduler
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    protocol: TCP
    port: 10251
    targetPort: 10251
    
# cat kube-scheduler-endpoints.yaml 
kind: Endpoints
apiVersion: v1
metadata:
  namespace: kube-system
  name: kube-scheduler
  labels:
    k8s-app: kube-scheduler
subsets:
  - addresses:
    - ip: 192.168.150.29
    ports:
    - name: http-metrics
      port: 10251
      protocol: TCP
```

- endpoints信息
```
# kubectl get endpoints 
NAME                      ENDPOINTS                                                                   AGE
fuseim.pri-ifs            <none>                                                                      112d
kube-controller           192.168.150.29:10252                                                        25m
kube-controller-manager   <none>                                                                      44m
kube-dns                  172.20.0.157:53,172.20.2.200:53,172.20.0.157:53 + 3 more...                 6h
kube-scheduler            192.168.150.29:10251                                                        112d
kubelet                   192.168.150.10:10255,192.168.150.29:10255,192.168.150.6:10255 + 6 more...   112d
kubernetes-dashboard      172.20.0.171:8443                                                           112d
metrics-server            172.20.0.172:443                                                            112d
tiller-deploy             172.20.2.187:44134                                                          57d

```