# Kubernetes NFS demo
The objective of this demonstration is to provide NFS mounts to a kubernetes pod while enforcing server-side permission write protections.
## Goals
1. Secure UID/GID on NFS
2. Provision NFS to a Pod
3. Additional options for discussion

## NFS Server-Side Setup
The demo will leverage Ubuntu 16.04 server providing NFS-Server services.
```
sudo apt-get install -y nfs-kernel-server
```
Let's configure the nfs export as /nfs/nfsvol1 along enforcing a generic users permissions. This configuration will enforce all writes to the nfs volume to this user on the server side. It may be a uid/gid of a real user, or further secured using nobody/nogroup. In this example, we'll use my uid/gid (1000) on the server side.
```
/nfs/nfsvol1 *(rw,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)
```
1. The volume "/nfs/nfsvol1" will be the local filesystem exported as a share
2. I'm using a wildcard here "*" to allow all remote mounts, however if you pin the pod to a specific node/ip then you can harden this point down as well.
3. Privileges are "rw" = read-write, "sync" = commit writes to disk before client acknowledgement, "no_subtree_check" = we're going to force permissions on writes, so I'll ignore these filesystem checks and improve performance, "all_sqash" = all uid's and gid's (including root) will be squashed (converted) on the nfs server side, "anonuid/anongid=1000" = this forces all users to write to the filesystem on the nfs server share as user 1000 (my uid/gid = shaker).

Let's create the share, set the permissions and export nfs.
1. mkdir /nfs/nfsvol1
2. chmod 775 /nfs/nfsvol1, this provides uid/gid write access all others read access.
3. chown shaker:shaker /nfs/nfsvol1, this sets the uid/gid permissions on the actual share.
4. exportfs -af, export the nfs filesystem

## Kubernetes Pod setup
This will comprise of 4 parts:
1. StorageClass, let's set NFS
2. PersistentVolume, provide the nfs-server mount point
3. PersistentVolumeClaim, provide a claim for the Pod to map to

## StorageClass = name: nfs
```
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: nfs
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
  labels:
    kubernetes.io/cluster-service: "true"
provisioner: kubernetes.io/nfs
```
## PersistentVolume = name: nfs-pv
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  nfs:
    path: /nfs/nfsvol1
    server: 172.16.1.5
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  ```
Above we're setting the nfs-server volume quota (1Gb), path (/nfs/nfsvol1) and location (172.16.1.5) by my nfs-server's IP address. For testing, I've chosen the "Retain" policy to preserve the data once the Pod is removed.

## PersistentVolumeClaim = name: nfs-pvc
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```
Above we set the nfs claim for the Pod.

## Pod = name: nginx-pod
I'm just using nginx as a simple pod that I can shell into and verify nfs; it doesn't have any other purpose.
```
kind: Pod
apiVersion: v1
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-pod
      image: nginx
      volumeMounts:
      - mountPath: "/opt"
        name: nfstest
  volumes:
    - name: nfstest
      persistentVolumeClaim:
        claimName: nfs-pvc
```
Above we have the Pod definition, again, we're using nginx only because it's an easy package for testing. The Pod will mount the nfs share at "/opt". 

### Mapping everything out, the Pod (nginx-pod) calls the PersistentVolumeClaim (nfs-pvc), the PersistentVolumeClaim refers to the StorageClass (nfs) so we know it's an nfs mount (kubernetes.io/nfs) and StorageClass (nfs) is also referenced by PersistenVolume so we bind the nfs-server mount point to the PersistentVolumeClaim.

Let's apply the yml
```
$ k apply -f mytestvol.yml
pod "nginx-pod" created
persistentvolumeclaim "nfs-pvc" created
persistentvolume "nfs-pv" created
storageclass.storage.k8s.io "nfs" created
```
Now let's exec into the pod and evaluate the filesystem
```
$ k exec -it nginx-pod sh
# df -h
Filesystem               Size  Used Avail Use% Mounted on
none                      50G  4.0G   46G   8% /
tmpfs                    3.9G     0  3.9G   0% /dev
tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
172.16.1.5:/nfs/nfsvol1   15G  2.0G   12G  15% /opt
/dev/sda1                 15G  2.3G   12G  16% /etc/hosts
/dev/sdb1                 50G  4.0G   46G   8% /etc/hostname
shm                       64M     0   64M   0% /dev/shm
tmpfs                    3.9G   12K  3.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                    3.9G     0  3.9G   0% /sys/firmware
tmpfs                    3.9G     0  3.9G   0% /proc/scsi
#
```
Above we can see the mount was successful as the nfs-server share is mapped correctly to /opt. Now let's touch some files and verify permissions.
```
# cd /opt
# touch a b c
# ls -l
total 0
-rw-r--r-- 1 1000 1000 0 Jun 22 17:19 a
-rw-r--r-- 1 1000 1000 0 Jun 22 17:19 b
-rw-r--r-- 1 1000 1000 0 Jun 22 17:19 c
```
The Pod has mapped the writes to UID/GID 1000 - as write+read+read, now let's review this from the nfs server.
```
shaker@nfs1:/nfs/nfsvol1$ ls -l
total 0
-rw-r--r-- 1 shaker shaker 0 Jun 22 13:19 a
-rw-r--r-- 1 shaker shaker 0 Jun 22 13:19 b
-rw-r--r-- 1 shaker shaker 0 Jun 22 13:19 c
```
The NFS Server also see's the same thing. We have successfully provisioned a NFS share to a Kubernetes Pod and restricted the permissions to UID/GID 1000 (shaker).


## Additional Options
Other uses would be to defined the UID/GID in the Pod and have it write to NFS as nobody:nogroup on the nfs-server. The thought behind all of this is to prevent any root level writes to occur on the share that could be compromising.

As I think of other uses and use cases, I'll add them to this section.

References:
1. Why change reclaim policy of a PersistentVolume
https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/

