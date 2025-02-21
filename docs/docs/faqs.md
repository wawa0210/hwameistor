---
sidebar_position: 8
sidebar_label: "FAQs"
---

# FAQs

## Q1: How does HwameiStor scheduler work in a Kubernetes platform?

The HwameiStor scheduler is deployed as a pod in the HwameiStor namespace.

![img](img/clip_image002.png)

Once the applications (Deployment or StatefulSet) are created, the pod will
be scheduled to the worker nodes on which HwameiStor is already configured.

## Q2: How does HwameiStor schedule applications with multi-replicas workloads and what are the differences compared to the traditional shared storage (NFS / block)?

We strongly recommend using StatefulSet for applications with multi-replica workloads.

StatefulSet will deploy replicas on the same worker node with the original pod, and will
also create a PV data volume for each replica. If you need to deploy replicas on different
worker nodes, you shall manually configure them with `pod affinity`.

![img](img/clip_image004.png)

We suggest using a single pod for deployment because the block data volumes can not be shared.

## Q3: How to maintain a Kubernetes node?

HwameiStor provides the volume eviction/migration functions to keep the Pods and HwameiStor
volumes' data running when retiring/rebooting a node.

### Remove a node

Before remove a node from a Kubernetes cluster, the Pods and volumes on the node should be
rescheduled and migrated to another available node, and keep the Pods/volumes running.

Follow these steps to remove a node:

1. Drain node.

    ```bash
    kubectl drain NODE --ignore-daemonsets=true. --ignore-daemonsets=true
    ```

    Allows the above command to succeed even if pods managed by daemonset exist.
    If it stacks due to PodDisruptionBudgets or something, try --force option.
    The command will also trigger HwameiStor to migrate all the volumes' replicas
    to another available node. Make sure the migration to complete by following
    command before moving ahead.

2. Check the migration progress.

    ```bash
    kubectl get localstoragenode NODE -o yaml
    ```

    The output may look like:

    ```yaml
    apiVersion: hwameistor.io/v1alpha1
    kind: LocalStorageNode
    metadata:
      name: NODE
    spec:
      hostname: NODE
      storageIP: 10.6.113.22
      topogoly:
        region: default
        zone: default
    status:
      ...
      pools:
        LocalStorage_PoolHDD:
          class: HDD
          disks:
          - capacityBytes: 17175674880
            devPath: /dev/sdb
            state: InUse
            type: HDD
          freeCapacityBytes: 16101933056
          freeVolumeCount: 999
          name: LocalStorage_PoolHDD
          totalCapacityBytes: 17175674880
          totalVolumeCount: 1000
          type: REGULAR
          usedCapacityBytes: 1073741824
          usedVolumeCount: 1
          volumeCapacityBytesLimit: 17175674880
          ## **** make sure volumes is empty **** ##
          volumes:  
      state: Ready
    ```

    At the same time, HwameiStor will automatically reschedule the evicted Pods
    to the other node which has the associated volume replica, and continue to run.

3. Remove the NODE from the cluster.

    ```bash
    kubectl delete nodes NODE
    ```

### Reboot a node

It usually takes a long time (~10minutes) to reboot a node. All the Pods and volumes on
the node will not work until the node is back online. For some applications like DataBase,
the long downtime is very costly and even unacceptable.

HwameiStor can immediately reschedule the Pod to another available node with associated
volume data and bring the Pod back to running in very short time (~ 10 seconds for the
Pod using a HA volume, and longer time for the Pod with non-HA volume depends on the data size).

1. Add a label (optional)

    If it is not required to migrate the volumes during the node reboots,
    you can add the following label to the node before draining it.

    ```bash
    kubectl label node NODE hwameistor.io/eviction=disable
    ```

2. Drain the node.

    ```bash
    kubectl drain NODE --ignore-daemonsets=true. --ignore-daemonsets=true
    ```

    - If Step 1 has been performed, you can reboot the node after Step 2 is successful.
    - If Step 1 has not been performed, you should check if the data migration is complete after Step 2 is successful (similar to Step 2 in [remove node](#remove-a-node)). After the data migration is complete, you can reboot the node.

    After the first two steps are successful, you can reboot the node and wait for the node system to return to normal.

3. Bring the node back to normal.

    ```bash
    kubectl uncordon NODE
    ```

### Traditional shared storage

In this case, StatefulSet will deploy replicas to other worker nodes for workload distribution and
will also create a PV data volume for each replica.

The `Deployment` will also deploy replicas to other worker nodes for workload distribution
but will share the same PV data volume (only for NFS). We suggest using a single pod for
block storage because the block data volumes can not be shared.
