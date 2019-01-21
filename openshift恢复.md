### openshift恢复

Restore an OpenShift cluster and its components by recreating cluster elements, including nodes and applications, from separate storage:

1. Reinstall OpenShift using the same method as previous installation (use the same inventory file).
2. Run any custom post-installation scripts.
3. Restore the node[s].

#### Restoring Masters

1. Export the backup directory:

   ```
   # MYBACKUPDIR=*/backup/$(hostname)/$(date +%Y%m%d)*
   ```

- Rename new `config`files created during installation:

  ```
  # cp /etc/origin/master/master-config.yaml /etc/origin/master/master-config.yaml.old
  ```

- Copy configuration files from backup`dir`to`/etc/origin/master/`:

  ```
  # cp /backup/$(hostname)/$(date +%Y%m%d)/origin/master/master-config.yaml /etc/origin/master/master-config.yaml
  ```

- Restart`origin-master-controllers`and`origin-master-api`services:

  ```
  # systemctl restart atomic-openshift-master-api \
    atomic-openshift-master-controllers
  ```

#### Restoring Nodes

1. Export the backup directory:

   ```
   # MYBACKUPDIR=*/backup/$(hostname)/$(date +%Y%m%d)* 
   ```

- Rename the`config`files created during installation:

  ```
  # cp /etc/origin/node/node-config.yaml \
       /etc/origin/node/node-config.yaml.old 
  ```

- Copy configuration files from backup to`/etc/origin/node/`:

  ```
  # cp /backup/$(hostname)/$(date +%Y%m%d)/etc/origin/node/node-config.yaml \
        /etc/origin/node/node-config.yaml 
  ```

- Restart`origin-node`service:

  ```shell
  # systemctl restart origin-node.service
  ```