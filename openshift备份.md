### openshift备份

Creating an environment-wide backup involves copying important data to assist with restoration in the case of crashing instances, or corrupt data. After backups have been created, they can be restored onto a newly installed version of the relevant component. In OpenShift, backups can be done at the cluster level for saving state to separate storage. The full state of an environment backup includes:

- Cluster data files
- etcd data on each master
- API objects
- Registry storage
- Volume storage

Node backups are optional, as the nature of nodes is that anything special is replicated over the nodes in case of failover, and they typically do not contain data that is necessary to run an environment. If nodes are configured with something that is necessary for the environment to remain operational, then creating a backup is recommended.

#### To manually backup a master:

```shell
master0 ~]# MYBACKUPDIR=/backup/$(hostname)/$(date +%Y%m%d)
master0 ~]# mkdir -p ${MYBACKUPDIR}/etc/sysconfig
master0 ~]#  cp -aR /etc/origin ${MYBACKUPDIR}/etc
master0 ~]# cp -aR /etc/sysconfig/origin-* ${MYBACKUPDIR}/etc/sysconfig/
master0 ~]# mkdir -p ${MYBACKUPDIR}/etc/sysconfig
master0 ~]# mkdir -p ${MYBACKUPDIR}/etc/pki/ca-trust/source/anchors
master0 ~]# cp -aR /etc/sysconfig/{iptables,docker-*} \ ${MYBACKUPDIR}/etc/sysconfig/
master0 ~]# cp -aR /etc/dnsmasq* /etc/cni ${MYBACKUPDIR}/etc/
master0 ~]# sudo cp -aR /etc/pki/ca-trust/source/anchors/* \ ${MYBACKUPDIR}/etc/pki/ca-trust/source/anchors/
```

#### To manually backup a node:

```shell
node0 ~]# MYBACKUPDIR=/backup/$(hostname)/$(date +%Y%m%d)

node0 ~]# mkdir -p ${MYBACKUPDIR}/etc/sysconfig

node0 ~]# mkdir -p ${MYBACKUPDIR}/etc/pki/ca-trust/source/anchors

node0 ~]# cp -aR /etc/sysconfig/{iptables,docker-*} \ ${MYBACKUPDIR}/etc/sysconfig/

node0 ~]# cp -aR /etc/dnsmasq* /etc/cni ${MYBACKUPDIR}/etc/

node0 ~]#  cp -aR /etc/pki/ca-trust/source/anchors/* \ ${MYBACKUPDIR}/etc/pki/ca-trust/source/anchors/
```

You can also use the **UNSUPPORTED** bash script provided in the [openshift-ansible-contrib](https://github.com/openshift/openshift-ansible-contrib.git)Github repository to create both master backups and project-level backups.