![argrocd-sync](https://argocd.doks.myshiny.space/api/badge?name=foodmag-app&revision=true)

# database-as-a-Service (DBaaS)
This repository content is used to illustrate the deployment or migration of a stateful/legacy application on Kubernetes (k8s) with persistent storage using [StorageOS](https://storageos.com) as cloud native stroage backend. The stateful application is composed of Drupal using PostgreSQL. 

The content is built as a step by step approach to grow knowledge from basic to advanced. The only knowledge requirements are basic linux and container with docker. 

## preparing the cluster
A good 50% of this repo could be done without having a real k8s cluster and using instead minikube or any similar one node k8s project. However, to make it through the full guide, it would be recommended, especially when looking into persistent storage.  

To deploy a k8s cluster, 3 VMs would be required, each having 2vCPU/2GB RAM/20GB disk. There are a couple of ways to deploy a 3 node k8s cluster without to much of hazzle like [k3s](https://k3s.io/) providing the user with a very lightweight k8s option consuming very little resources but yet offering the basic magic power of k8s.

For the purposes of this guide, a k8s cluster deployed within DigitalOcean is used. The overall guide should be agnostic of any k8s cluster provider (aks, eks, ...). 

### connecting to k8s
The first component to connect to k8s platform is [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/), the link provides the installation methodology based on the operating system.  
The second component to connect is a config file also referred as kubeconfig which presents itself as yaml file. If on linux machine, the file is usually found in: 
``` 
ls -al ~/.kube/
```
The file can save in that path under the name config or any other that can be exported as an environment variable as such:
```
export KUBECONFIG=~/.kube/file_name 
```
like so if the file was called ```digitalocean```: 
```
export KUBECONFIG=~/.kube/digitalocean
```
Once exported, ```kubectl``` can be used to verify connectivity:
```
kubectl get nodes
NAME          STATUS   ROLES    AGE   VERSION
dbaas-8row6   Ready    <none>   54m   v1.20.2
dbaas-8rowa   Ready    <none>   54m   v1.20.2
dbaas-8rowe   Ready    <none>   54m   v1.20.2
```
This output shows a 3 node cluster running a kubernetes version 1.20.2 freshly deployed ready to schedule workload.

### deploying a persistent storage solution
K8s is designed to support stateless workload natively. To support stateful workload, a persistent storage solution has to be implemented in order to provide a volume to be consummed by the application. The volume will survice the application rescheduling, scaling up/down, or deletion.  
[StorageOS](https://storageos.com) is cloud native, software-defined storage for running containerized applications in production, running in the cloud, on-prem and in hybrid/multi-cloud environments. 

StorageOS has a free forever developer tier that can be used for such environment or for a development platform. A [self evaluation guide](https://docs.storageos.com/docs/self-eval/) provides the user with a script to do a rapid deployment which will consume local disk space on nodes while providing a fully distributed persistent storage for stateful applications. Remember, this is not suited for a production deployment but performed for testing. 

From the admin machine where ```kubectl``` is connected with freshly deployed k8s cluster, the following script can be ran:
```
curl -sL https://storageos.run | bash
```
Wait! Be brave, don't just copy/paste the above... curl first, review and then execute.

The output would like like:
```
rovandep in ~ ❯ curl -sL https://storageos.run |bash
Welcome to the StorageOS quick installation script.
I will install StorageOS version v2.4.0-rc.1 into
namespace kube-system now. If I encounter any errors
I will stop immediately.

Creating etcd namespace storageos-etcd
namespace/storageos-etcd created
Creating etcd ClusterRole and ClusterRoleBinding
clusterrolebinding.rbac.authorization.k8s.io/etcd-operator created
clusterrole.rbac.authorization.k8s.io/etcd-operator created
Creating etcd operator Deployment
deployment.apps/etcd-operator created
Creating etcd cluster in namespace storageos-etcd
etcdcluster.etcd.database.coreos.com/storageos-etcd created
Installing StorageOS Operator version v2.4.0-rc.1
Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 CustomResourceDefinition
customresourcedefinition.apiextensions.k8s.io/storageosclusters.storageos.com created
customresourcedefinition.apiextensions.k8s.io/storageosupgrades.storageos.com created
customresourcedefinition.apiextensions.k8s.io/jobs.storageos.com created
customresourcedefinition.apiextensions.k8s.io/nfsservers.storageos.com created
namespace/storageos-operator created
clusterrole.rbac.authorization.k8s.io/storageos-operator created
serviceaccount/storageoscluster-operator-sa created
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRoleBinding
clusterrolebinding.rbac.authorization.k8s.io/storageoscluster-operator-rolebinding created
deployment.apps/storageos-cluster-operator created
Operator installed, waiting for pod to become ready
StorageOS Operator installed successfully
Creating Secret definining the API Username and Password
secret/storageos-api created
Installing StorageOS Cluster version v2.4.0-rc.1
storageoscluster.storageos.com/self-evaluation created
Waiting for StorageOS pods to become ready
Waiting for StorageOS pods to become ready
Waiting for StorageOS pods to become ready
Waiting for StorageOS pods to become ready
Waiting for StorageOS pods to become ready
StorageOS Cluster installed successfully
Deploying the StorageOS CLI as a pod in the kube-system namespace
pod/cli created
Waiting for the cli pod to become ready
StorageOS CLI pod is running
Your StorageOS Cluster now is up and running!

Now would be a good time to deploy your first volume - see
https://docs.storageos.com/docs/self-eval/#a-namestorageosvolumeaprovision-a-storageos-volume
for an example of how to mount a StorageOS volume in a pod

Don't forget to license your cluster - see https://docs.storageos.com/docs/operations/licensing/

This cluster has been set up with an etcd based on ephemeral
storage. It is suitable for evaluation purposes only - for
production usage please see our etcd installation nodes at
https://docs.storageos.com/docs/prerequisites/etcd/
```

Let's verify the status of the StorageOS cluster within our k8s cluster:
```
kubectl exec -n kube-system -it cli -- storageos get cluster
ID:           ed08d40e-5035-4270-81bd-2af74d53b5f8
Created at:   2021-05-07T09:32:40Z (2 minutes ago)
Updated at:   2021-05-07T09:32:40Z (2 minutes ago)
Nodes:        3
  Healthy:    3
  Unhealthy:  0
```

## my first app
### show me the YAML
Before starting to run, let's have a little walk. The following example is from the actual StorageOS self evaluation guide providing two YAML definition for creating a Persistent Volume Claim (PVC) and an application (Pod) that will consume the persistent volume (PV). 

```myfirstpvc.yaml```:
```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-1
spec:
  storageClassName: "fast"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

The above provides a "voucher" with the name "pvc-1" for the application to get a volume of 5GB out of the storage class called "fast". This would be the most simplistic attempt to explain a PVC.  

```myfirstpod.yaml```: 
```yaml
---
apiVersion: v1
kind: Pod
metadata:
 name: d1
spec:
 containers:
   - name: debian
     image: debian:9-slim
     command: ["/bin/sleep"]
     args: [ "3600" ]
     volumeMounts:
       - mountPath: /mnt
         name: v1
 volumes:
   - name: v1
     persistentVolumeClaim:
       claimName: pvc-1
```

The above provides a blueprint of our application. It tells that we will be getting a debian container base image with a specific nature (9-slim), that we will run a command once up and that we want to redeem our volume voucher and mounting the volume at ```/mnt```.

Notes:
- note the comment structure of the descriptive configuration file. This is a well-documented standard. 
- two files are created but both YAML code could be append within the same file based on the above output. 

### show the running YAML
To actually deploy the first app configuration, the following command can be executed:

```
kubectl apply -f myfirstapp/myfirstpvc.yaml
persistentvolumeclaim/pvc-1 created
kubectl apply -f myfirstapp/myfirstpod.yaml
pod/d1 created
```
Wow! no fireworks or music? nope... it just did it! 

The results will be two objects that are linked together:
```
kubectl get pvc
NAME    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-1   Bound    pvc-f4af80a7-1224-4641-abae-8403e3c9827b   5Gi        RWO            fast           86s
kubectl get pod
NAME   READY   STATUS    RESTARTS   AGE
d1     1/1     Running   0          95s
```

### illustrate persistent storage
The goal of this first app is to provide an undestanding of the different objects like PVC, PV, or Pod and then illustrate the ephemeral nature of a Pod. 

Let's connect to the running pod and save some important message on our volume.
```
kubectl exec -it d1 -- bash
root@d1:/# mount |grep mnt
/var/lib/storageos/volumes/v.692c7205-3fab-4e37-969f-27e0b0268776 on /mnt type ext4 (rw,relatime,stripe=32)
root@d1:/# echo k8s rocks! > /mnt/motd
root@d1:/# cat /mnt/motd
k8s rocks!
root@d1:/# cat /etc/motd

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@d1:/# echo k8s rocks! > /etc/motd
root@d1:/# cat /etc/motd
k8s rocks!
root@d1:/# exit
```

The important message has been saved at two different places:
- the famous ```/etc``` directory where the motd file would be expected, which is now modified with our important message
- the ```/mnt``` directory where the persistent volume is mounted

Let's destroy the Pod and observe the container image ephemeral nature:
```
kubectl delete pod d1
pod "d1" deleted
kubectl get pod
No resources found in default namespace.
kubectl get pvc
NAME    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-1   Bound    pvc-f4af80a7-1224-4641-abae-8403e3c9827b   5Gi        RWO            fast           13m
```
The above output confirms that the Pod is delete but the PVC and volume still exist. The beauty of Kubernetes and it's descriptive desired state configuration file is that it can be applied again and the very same Pod will start and attach to the volume:

```
kubectl apply -f myfirstapp/myfirstpod.yaml
pod/d1 created
rovandep in DBaaS on  main ↑3 ↓2  ~1 ❯ kubectl get pod
NAME   READY   STATUS              RESTARTS   AGE
d1     0/1     ContainerCreating   0          4s
kubectl get pod
NAME   READY   STATUS    RESTARTS   AGE
d1     1/1     Running   0          12s
```

Let's check on our motd:

``` 
 kubectl exec -it d1 -- bash
root@d1:/# mount |grep mnt
/var/lib/storageos/volumes/v.692c7205-3fab-4e37-969f-27e0b0268776 on /mnt type ext4 (rw,relatime,stripe=32)
root@d1:/# cat /mnt/motd
k8s rocks!
root@d1:/# cat /etc/motd

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@d1:/# exit
```

The above output confirms that a container image is indeed ephemeral and will discard any changes when deleted and redeployed versus having a persistent storage that can keep data across multiple scenarios from life-cycle, redeployment or failures.

If a PVC is a voucher for a volume, a container image is an ISO used for a LiveCD operating system, the Pod, and no data being generated during the usage of the LiveCD will be kept and written on the ISO for later on.

