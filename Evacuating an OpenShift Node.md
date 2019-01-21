# Evacuating an OpenShift Node

Evacuating pods allows you to migrate all or selected pods from a given node or nodes, but nodes must first be marked `unschedulable` to perform pod evacuation.

Only pods backed by a replication controller can be evacuated; the replication controllers create new pods on other nodes and remove the existing pods from the specified node(s). Bare pods, meaning those not backed by a replication controller, are unaffected by default.

To perform a node evacuation:

1. First, mark the decommissioned node as`unschedulable`using the OpenShift Client:

   ```
   $ oc adm manage-node $NODE_NAME --schedulable=false
   ```

- Once the node is marked as`SchedulingDisabled`, evacuate pods to other non-infra and non-master nodes using OpenShift Clients:

  ```
  $ oc adm drain $NODE_NAME
  ```

