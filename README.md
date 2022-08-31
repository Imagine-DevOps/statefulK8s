# Configuring and managing persistent storage

K8s storage solutions that protect state of apps from node/app failures, data sharing, and volume reattachment when pod is rescheduled on a different node.

### Technical Requirements:  

Kubernetes' main command-line tool, kubectl , also use helm where helm charts are available to deploy solutions.

These recipes allow the flexibility of utilizing multiple storage providers using popular storage orchestrator for your applications.

## 1st. Recipe - Rook
-   Installing a Ceph provider using Rook
-   Creating a Ceph cluster
-   Verifying a Ceph cluster's health
-   Create a Ceph block storage class
-   Using a Ceph block storage class to create dynamic PVs

### Installing a Ceph provider using Rook

Let's perform the following steps to get a Ceph scale-out storage solution up and running using the Rook project:

1.  Clone the Rook repository:

```markup
$ git clone https://github.com/rook/rook.git$ cd rook/cluster/examples/kubernetes/ceph/
```

2.  Deploy the Rook Operator:

```markup
$ kubectl create -f common.yaml $ kubectl create -f operator.yaml 
```

3.  Verify the Rook Operator:

```markup
$ kubectl get pod -n rook-cephNAME                                READY STATUS  RESTARTS AGErook-ceph-operator-6b66859964-vnrfx 1/1   Running 0        2m12srook-discover-8snpm                 1/1   Running 0        97srook-discover-mcx9q                 1/1   Running 0        97srook-discover-mdg2s                 1/1   Running 0        97s
```

Now you have learned how to deploy the Rook orchestration components for the Ceph provider running on Kubernetes.

### Creating a Ceph cluster

Let's perform the following steps to deploy a Ceph cluster using the Rook Operator:

1.  Create a Ceph cluster:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: ceph.rook.io/v1kind: CephClustermetadata:  name: rook-ceph  namespace: rook-cephspec:  cephVersion:    image: ceph/ceph:v14.2.3-20190904  dataDirHostPath: /var/lib/rook  mon:    count: 3  dashboard:    enabled: true  storage:    useAllNodes: true    useAllDevices: false    directories:    - path: /var/lib/rookEOF
```

2.  Verify that all pods are running:

```markup
$ kubectl get pod -n rook-ceph
```

Within a minute, a fully functional Ceph cluster will be deployed and ready to be used. You can read more about Ceph in the _Rook Ceph Storage Documentation_ link in the _See also_ section.

### Verifying a Ceph cluster's health

The Rook toolbox is a container with common tools used for rook debugging and testing. Let's perform the following steps to deploy the Rook toolbox to verify cluster health:

1.  Deploy the Rook toolbox:

```markup
$ kubectl apply -f toolbox.yaml
```

2.  Verify that the toolbox is running:

```markup
$ kubectl -n rook-ceph get pod -l "app=rook-ceph-tools"NAME                             READY STATUS  RESTARTS AGErook-ceph-tools-6fdfc54b6d-4kdtm 1/1   Running 0        109s
```

3.  Connect to the toolbox:

```markup
$ kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
```

4.  Verify that the cluster is in a healthy state (HEALTH\_OK):

```markup
# ceph status  cluster:    id: 6b6e4bfb-bfef-46b7-94bd-9979e5e8bf04    health: HEALTH_OK  services:    mon: 3 daemons, quorum a,b,c (age 12m)    mgr: a(active, since 12m)    osd: 3 osds: 3 up (since 11m), 3 in (since 11m)  data:    pools: 0 pools, 0 pgs    objects: 0 objects, 0 B    usage: 49 GiB used, 241 GiB / 291 GiB avail    pgs:
```

5.  When you are finished troubleshooting, remove the deployment using the following command:

```markup
$ kubectl -n rook-ceph delete deployment rook-ceph-tools
```

Now you know how to deploy the Rook toolbox with its common tools that are used to debug and test Rook.

### Create a Ceph block storage class

Let's perform the following steps to create a storage class for Ceph storage.:

1.  Create CephBlockPool:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: ceph.rook.io/v1kind: CephBlockPoolmetadata:  name: replicapool  namespace: rook-cephspec:  failureDomain: host  replicated:    size: 3EOF
```

2.  Create a Rook Ceph block storage class:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: storage.k8s.io/v1kind: StorageClassmetadata:   name: rook-ceph-blockprovisioner: rook-ceph.rbd.csi.ceph.comparameters:    clusterID: rook-ceph    pool: replicapool    imageFormat: "2"    imageFeatures: layering    csi.storage.k8s.io/provisioner-secret-name: rook-ceph-csi    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph    csi.storage.k8s.io/node-stage-secret-name: rook-ceph-csi    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph    csi.storage.k8s.io/fstype: xfsreclaimPolicy: DeleteEOF
```

3.  Confirm that the storage class has been created:

```markup
$ kubectl get scNAME              PROVISIONER                AGEdefault (default) kubernetes.io/azure-disk   6h27mrook-ceph-block   rook-ceph.rbd.csi.ceph.com 3s
```

As you can see from the preceding provisioner name, rook-ceph.rbd.csi.ceph.com, Rook also uses CSI to interact with Kubernetes APIs. This driver is optimized for RWO pod access where only one pod may access the storage.

### Using a Ceph block storage class to create dynamic PVs

In this recipe, we will deploy Wordpress using dynamic persistent volumes created by the Rook Ceph block storage provider. Let's perform the following steps:

1.  Clone the examples repository:

```markup
$ git clone https://github.com/k8sdevopscookbook/src.git$ cd src/chapter5/rook/
```

2.  Review both mysql.yaml and wordpress.yaml. Note that PVCs are using the rook-ceph-block storage class:

```markup
$ cat mysql.yaml && cat wordpress.yaml
```

3.  Deploy MySQL and WordPress:

```markup
$ kubectl apply -f mysql.yaml $ kubectl apply -f wordpress.yaml
```
# Setting up NFS for shared storage on Kubernetes

Note: NFS is still used with cloud-native applications where multi-node write access is required. OpenEBS and Rook to ReadWriteMany (RWX) accessible persistent volumes for stateful workloads that require shared storage on Kubernetes.

## Technical Requirements

For this recipe, we need to have either rook or openebs installed as an orchestrator. Make sure that you have a Kubernetes cluster ready and kubectl configured to manage the cluster resources.



4.  Confirm the persistent volumes created:

```markup
$ kubectl get pvNAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGEpvc-eb2d23b8-d38a-11e9-88a2-a2c82783dcda 20Gi RWO Delete Bound default/mysql-pv-claim rook-ceph-block 38spvc-eeab1ebc-d38a-11e9-88a2-a2c82783dcda 20Gi RWO Delete Bound default/wp-pv-claim rook-ceph-block 38s
```

5.  Get the external IP of the WordPress service:

```markup
$ kubectl get serviceNAME            TYPE         CLUSTER-IP  EXTERNAL-IP  PORT(S)      AGEkubernetes      ClusterIP    10.0.0.1    <none>       443/TCP      6h34mwordpress       LoadBalancer 10.0.102.14 13.64.96.240 80:30596/TCP 3m36swordpress-mysql ClusterIP    None        <none>       3306/TCP     3m42s
```

6.  Open the external IP of the WordPress service in your browser to access your Wordpress deployment:


## 2nd. RECIPE OpenEBS - popular cloud-native storage project

-   Installing iSCSI client prerequisites
-   Installing OpenEBS
-   Using ephemeral storage to create persistent volumes
-   Creating storage pools
-   Creating OpenEBS storage classes
-   Using an OpenEBS storage class to create dynamic PVs

### Installing iSCSI client prerequisites

The OpenEBS storage provider requires that the iSCSI client runs on all worker nodes:

1.  On all your worker nodes, follow the steps to install and enable open-iscsi:

```markup
$ sudo apt-get update && sudo apt-get install open-iscsi && sudo service open-iscsi restart
```

2.  Validate that the iSCSI service is running:

```markup
$ systemctl status iscsid● iscsid.service - iSCSI initiator daemon (iscsid)   Loaded: loaded (/lib/systemd/system/iscsid.service; enabled; vendor preset: enabled)   Active: active (running) since Sun 2019-09-08 07:40:43 UTC; 7s ago     Docs: man:iscsid(8)
```

3.  If the service status is showing as inactive, then enable and start the iscsid service:

```markup
$ sudo systemctl enable iscsid && sudo systemctl start iscsid
```

After installing the iSCSI service, you are ready to install OpenEBS on your cluster.

### Installing OpenEBS

Let's perform the following steps to quickly get the OpenEBS control plane installed:

1.  Install OpenEBS services by using the operator:

```markup
$ kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
```

2.  Confirm that all OpenEBS pods are running:

```markup
$ kubectl get pods --namespace openebsNAME                                        READY STATUS  RESTARTS AGEmaya-apiserver-dcbc87f7f-k99fz              0/1   Running 0        88sopenebs-admission-server-585c6588d-j29ng    1/1   Running 0        88sopenebs-localpv-provisioner-cfbd49877-jzjxl 1/1   Running 0        87sopenebs-ndm-fcss7                           1/1   Running 0        88sopenebs-ndm-m4qm5                           1/1   Running 0        88sopenebs-ndm-operator-bc76c6ddc-4kvxp        1/1   Running 0        88sopenebs-ndm-vt76c                           1/1   Running 0        88sopenebs-provisioner-57bbbd888d-jb94v        1/1   Running 0        88sopenebs-snapshot-operator-7dd598c655-2ck74  2/2   Running 0        88s
```

OpenEBS consists of the core components listed here. Node Disk Manager (NDM) is one of the important pieces of OpenEBS that is responsible for detecting disk changes and runs as DaemonSet on your worker nodes.

### Using ephemeral storage to create persistent volumes

OpenEBS currently provides three storage engine options (Jiva, cStor, and LocalPV). The first storage engine option, Jiva, can create replicated storage on top of the ephemeral storage. Let's perform the following steps to get storage using ephemeral storage configured:

1.  List the default storage classes:

```markup
$ kubectl get scNAME                      PROVISIONER                                              AGEopenebs-device            openebs.io/local                                         25mopenebs-hostpath          openebs.io/local                                         25mopenebs-jiva-default      openebs.io/provisioner-iscsi                             25mopenebs-snapshot-promoter volumesnapshot.external-storage.k8s.io/snapshot-promoter 25m
```

2.  Describe the openebs-jiva-default storage class:

```markup
$ kubectl describe sc openebs-jiva-defaultName: openebs-jiva-defaultIsDefaultClass: NoAnnotations: cas.openebs.io/config=- name: ReplicaCount  value: "3"- name: StoragePool  value: default
```

3.  Create a persistent volume claim using openebs-jiva-default:

```markup
$ cat <<EOF | kubectl apply -f -kind: PersistentVolumeClaimapiVersion: v1metadata:  name: demo-vol1-claimspec:  storageClassName: openebs-jiva-default  accessModes:    - ReadWriteOnce  resources:    requests:      storage: 5GEOF
```

4.  Confirm that the PVC status is BOUND:

```markup
$ kubectl get pvcNAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGEdemo-vol1-claim Bound pvc-cb7485bc-6d45-4814-adb1-e483c0ebbeb5 5G RWO openebs-jiva-default 4s
```

5.  Now, use the PVC to dynamically provision a persistent volume:

```markup
$ kubectl apply -f https://raw.githubusercontent.com/openebs/openebs/master/k8s/demo/percona/percona-openebs-deployment.yaml
```

6.  Now list the pods and make sure that your workload, OpenEBS controller, and replicas are all in the running state:

```markup
$ kubectl get podsNAME                                                          READY STATUS  RESTARTS AGEpercona-767db88d9d-2s8np                                      1/1   Running 0        75spvc-cb7485bc-6d45-4814-adb1-e483c0ebbeb5-ctrl-54d7fd794-s8svt 2/2   Running 0        2m23spvc-cb7485bc-6d45-4814-adb1-e483c0ebbeb5-rep-647458f56f-2b9q4 1/1   Running 1        2m18spvc-cb7485bc-6d45-4814-adb1-e483c0ebbeb5-rep-647458f56f-nkbfq 1/1   Running 0        2m18spvc-cb7485bc-6d45-4814-adb1-e483c0ebbeb5-rep-647458f56f-x7s9b 1/1   Running 0        2m18s
```

Now you know how to get highly available, cloud-native storage configured for your stateful applications on Kubernetes.

### Creating storage pools

In this recipe, we will use raw block devices attached to your nodes to create a storage pool. These devices can be AWS EBS volumes, GCP PDs, Azure Disk, virtual disks, or vSAN volumes. Devices can be attached to your worker node VMs, or basically physical disks if you are using a bare-metal Kubernetes cluster. Let's perform the following steps to create a storage pool out of raw block devices:

1.  List unused and unclaimed block devices on your nodes:

```markup
$ kubectl get blockdevices -n openebsNAME NODENAME SIZE CLAIMSTATE STATUS AGEblockdevice-24d9b7652893384a36d0cc34a804c60c ip-172-23-1-176.us-west-2.compute.internal 107374182400 Unclaimed Active 52sblockdevice-8ef1fd7e30cf0667476dba97975d5ac9 ip-172-23-1-25.us-west-2.compute.internal 107374182400 Unclaimed Active 51sblockdevice-94e7c768ef098a74f3e2c7fed6d82a5f ip-172-23-1-253.us-west-2.compute.internal 107374182400 Unclaimed Active 52s
```

In our example, we have a three-node Kubernetes cluster on AWS EC2 with one additional EBS volume attached to each node.

2.  Create a storage pool using the unclaimed devices from _Step 1_:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: openebs.io/v1alpha1kind: StoragePoolClaimmetadata:  name: cstor-disk-pool  annotations:    cas.openebs.io/config: |      - name: PoolResourceRequests        value: |-            memory: 2Gi      - name: PoolResourceLimits        value: |-            memory: 4Gispec:  name: cstor-disk-pool  type: disk  poolSpec:    poolType: striped  blockDevices:    blockDeviceList:    - blockdevice-24d9b7652893384a36d0cc34a804c60c    - blockdevice-8ef1fd7e30cf0667476dba97975d5ac9    - blockdevice-94e7c768ef098a74f3e2c7fed6d82a5fEOF
```

3.  List the storage pool claims:

```markup
$ kubectl get spcNAME AGEcstor-disk-pool 29s
```

4.  Verify that a cStor pool has been created and that its status is Healthy:

```markup
$ kubectl get cspNAME                 ALLOCATED FREE  CAPACITY STATUS  TYPE    AGEcstor-disk-pool-8fnp 270K      99.5G 99.5G    Healthy striped 3m9scstor-disk-pool-nsy6 270K      99.5G 99.5G    Healthy striped 3m9scstor-disk-pool-v6ue 270K      99.5G 99.5G    Healthy striped 3m10s
```

5.  Now we can use the storage pool in storage classes to provision dynamic volumes.

### Creating OpenEBS storage classes

Let's perform the following steps to create a new storage class to consume StoragePool, which we created previously in the _Creating storage pools_ recipe:

1.  Create an OpenEBS cStor storage class using the cStor StoragePoolClaim name, cstor-disk-pool, with three replicas:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: storage.k8s.io/v1kind: StorageClassmetadata:  name: openebs-cstor-default  annotations:    openebs.io/cas-type: cstor    cas.openebs.io/config: |      - name: StoragePoolClaim        value: "cstor-disk-pool"      - name: ReplicaCount        value: "3"provisioner: openebs.io/provisioner-iscsiEOF
```

2.  List the storage classes:

```markup
$ kubectl get scNAME                  PROVISIONER                  AGEdefault               kubernetes.io/aws-ebs        25mgp2 (default)         kubernetes.io/aws-ebs        25mopenebs-cstor-default openebs.io/provisioner-iscsi 6sopenebs-device        openebs.io/local             20mopenebs-hostpath      openebs.io/local             20mopenebs-jiva-default  openebs.io/provisioner-iscsi 20mopenebs-snapshot-promoter volumesnapshot.external-storage.k8s.io/snapshot-promoter 20mubun
```

3.  Set the gp2 AWS EBS storage class as the non-default option:

```markup
$ kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.beta.kubernetes.io/is-default-class":"false"}}}'
```

4.  Define openebs-cstor-default as the default storage class:

```markup
$ kubectl patch storageclass openebs-cstor-default -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Make sure that the previous storage class is no longer set as the default and that you only have one default storage class.

### Using an OpenEBS storage class to create dynamic PVs

Let's perform the following steps to deploy dynamically created persistent volumes using the OpenEBS storage provider:

1.  Clone the examples repository:

```markup
$ git clone https://github.com/k8sdevopscookbook/src.git$ cd src/chapter5/openebs/
```

2.  Review minio.yaml and note that PVCs are using the openebs-stor-default storage class.

3.  Deploy Minio:

```markup
$ kubectl apply -f minio.yamldeployment.apps/minio-deployment createdpersistentvolumeclaim/minio-pv-claim createdservice/minio-service created
```

4.  Get the Minio service load balancer's external IP:

```markup
$ kubectl get serviceNAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGEkubernetes ClusterIP 10.3.0.1 <none> 443/TCP 54mminio-service LoadBalancer 10.3.0.29 adb3bdaa893984515b9527ca8f2f8ca6-1957771474.us-west-2.elb. amazonaws.com 9000:32701/TCP 3s
```

5.  Add port 9000 to the end of the address and open the external IP of the Minio service in your browser:
6.  Use the username minio, and the password minio123 to log in to the Minio deployment backed by persistent OpenEBS volumes:

![](https://static.packt-cdn.com/products/9781838828042/graphics/assets/04aacbb2-701d-474e-bc57-dd66a0ae32f3.png)

You have now successfully deployed a stateful application that is deployed on the OpenEBS cStor storage engine.

# Setting up NFS for shared storage on Kubernetes

# NFS is still used with cloud-native applications where multi-node write access is required. OpenEBS and Rook to ReadWriteMany (RWX) accessible persistent volumes for stateful workloads that require shared storage on Kubernetes.



-   Installing NFS prerequisites
-   Installing an NFS provider using a Rook NFS operator
-   Using a Rook NFS operator storage class to create dynamic NFS PVs
-   Installing an NFS provider using OpenEBS
-   Using the OpenEBS operator storage class to create dynamic NFS PVs

### Installing NFS prerequisites

To be able to mount NFS volumes, NFS client packages need to be preinstalled on all worker nodes where you plan to have NFS-mounted pods:

1.  If you are using Ubuntu, install nfs-common on all worker nodes:

```markup
$ sudo apt install -y nfs-common
```

2.  If using CentOS, install nfs-common on all worker nodes:

```markup
$ yum install nfs-utils
```

Now we have nfs-utils installed on our worker nodes and are ready to get the NFS server to deploy.

### Installing an NFS provider using a Rook NFS operator

Let's perform the following steps to get an NFS provider functional using the Rook NFS provider option:

1.  Clone the Rook repository:

```markup
$ git clone https://github.com/rook/rook.git$ cd rook/cluster/examples/kubernetes/nfs/
```

2.  Deploy the Rook NFS operator:

```markup
$ kubectl create -f operator.yaml
```

3.  Confirm that the operator is running:

```markup
$ kubectl get pods -n rook-nfs-systemNAME READY STATUS RESTARTS AGErook-nfs-operator-54cf68686c-f66f5 1/1 Running 0 51srook-nfs-provisioner-79fbdc79bb-hf9rn 1/1 Running 0 51s
```

4.  Create a namespace, rook-nfs:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: v1kind: Namespacemetadata: name: rook-nfsEOF
```

5.  Make sure that you have defined your preferred storage provider as the default storage class. In this recipe, we are using openebs-cstor-default, defined in persistent storage using the OpenEBS recipe.
6.  Create a PVC:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: v1kind: PersistentVolumeClaimmetadata:  name: nfs-default-claim  namespace: rook-nfsspec:  accessModes:  - ReadWriteMany  resources:    requests:      storage: 1GiEOF
```

7.  Create the NFS instance:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: nfs.rook.io/v1alpha1kind: NFSServermetadata:  name: rook-nfs  namespace: rook-nfsspec:  serviceAccountName: rook-nfs  replicas: 1  exports:  - name: share1    server:      accessMode: ReadWrite      squash: "none"    persistentVolumeClaim:      claimName: nfs-default-claim  annotations:  # key: valueEOF
```

8.  Verify that the NFS pod is in the Running state:

```markup
$ kubectl get pod -l app=rook-nfs -n rook-nfs NAME       READY  STATUS  RESTARTS AGErook-nfs-0 1/1    Running 0        2m
```

By observing the preceding command, an NFS server instance type will be created.

### Using a Rook NFS operator storage class to create dynamic NFS PVs

NFS is used in the Kubernetes environment on account of its ReadWriteMany capabilities for the application that requires access to the same data at the same time. In this recipe, we will perform the following steps to dynamically create an NFS-based persistent volume:

1.  Create Rook NFS storage classes using exportName, nfsServerName, and nfsServerNamespace from the _Installing an NFS provider using a Rook NFS operator_ recipe:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: storage.k8s.io/v1kind: StorageClassmetadata:  labels:    app: rook-nfs  name: rook-nfs-share1parameters:  exportName: share1  nfsServerName: rook-nfs  nfsServerNamespace: rook-nfsprovisioner: rook.io/nfs-provisionerreclaimPolicy: DeletevolumeBindingMode: ImmediateEOF
```

2.  Now, you can use the rook-nfs-share1 storage class to create PVCs for applications that require ReadWriteMany access:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: v1kind: PersistentVolumeClaimmetadata:  name: rook-nfs-pv-claimspec:  storageClassName: "rook-nfs-share1"  accessModes:    - ReadWriteMany  resources:    requests:      storage: 1MiEOF
```

By observing the preceding command, an NFS PV will be created.

### Installing an NFS provisioner using OpenEBS

OpenEBS provides an NFS provisioner that is protected by the underlying storage engine options of OpenEBS. Let's perform the following steps to get an NFS service with OpenEBS up and running:

1.  Clone the examples repository:

```markup
$ git clone https://github.com/k8sdevopscookbook/src.git$ cd src/chapter5/openebs
```

2.  In this recipe, we are using the openebs-jiva-default storage class. Review the directory content and apply the YAML file under the NFS directory:

```markup
$ kubectl apply -f nfs
```

3.  List the PVCs and confirm that a PVC named openebspvc has been created:

```markup
$ kubectl get pvcNAME       STATUS VOLUME                                   CAPACITY ACCESS MODES STORAGECLASS         AGEopenebspvc Bound  pvc-9f70c0b4-efe9-4534-8748-95dba05a7327 110G     RWO          openebs-jiva-default 13m
```

### Using the OpenEBS NFS provisioner storage class to create dynamic NFS PVs

Let's perform the following steps to dynamically deploy an NFS PV protected by the OpenEBS storage provider:

1.  List the storage classes, and confirm that openebs-nfs exists:

```markup
$ kubectl get scNAME                            PROVISIONER                  AGEopenebs-cstor-default (default) openebs.io/provisioner-iscsi 14hopenebs-device                  openebs.io/local             15hopenebs-hostpath                openebs.io/local             15hopenebs-jiva-default            openebs.io/provisioner-iscsi 15hopenebs-nfs                     openebs.io/nfs               5sopenebs-snapshot-promoter       volumesnapshot.external-storage.k8s.io/snapshot-promoter 15h
```

2.  Now, you can use the openebs-nfs storage class to create PVCs for applications that require ReadWriteMany access:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: v1kind: PersistentVolumeClaimmetadata:  name: openebs-nfs-pv-claimspec:  storageClassName: "openebs-nfs"  accessModes:    - ReadWriteMany  resources:    requests:      storage: 1MiEOF
```

## See also

-   Rook NFS operator documentation: [https://github.com/rook/rook/blob/master/Documentation/nfs.md](https://github.com/rook/rook/blob/master/Documentation/nfs.md)
-   OpenEBS provisioning read-write-many PVCs: [https://docs.openebs.io/docs/next/rwm.html](https://docs.openebs.io/docs/next/rwm.html)
