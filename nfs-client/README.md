# Dynamic Kubernetes NFS-Client Provisioner

[![Docker Repository on Quay](https://quay.io/repository/external_storage/nfs-client-provisioner/status "Docker Repository on Quay")](https://quay.io/repository/external_storage/nfs-client-provisioner)


`nfs-client` is an automatic provisioner that used your *already configured* NFS server, automatically creating Persistent Volumes.

- Persistent volumes are provisioned as ${namespace}-${pvcName}-${pvName}
- Persistent volumes which are recycled as archieved-${namespace}-${pvcName}-${pvName}

# How to deploy nfs-client to your cluster.

To note, you must *already* have an NFS Server.

1. Editing:

Modify `deployment.yaml` and change the values to your own NFS server:


```yaml
          env:
            - name: PROVISIONER_NAME
              value: auto-nfs
            - name: NFS_SERVER
              value: 172.16.1.25
            - name: NFS_PATH
              value: /nfs/vol1
      volumes:
        - name: nfs-client-root
          nfs:
            server: 172.16.1.25
            path: /nfs/vol1
```

Modify `class.yaml` to match the same value indicated by `PROVISIONER_NAME`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: auto-nfs
# or choose another name, must match deployment's env PROVISIONER_NAME'
```

2. Authorization - Docker EE 2.0 RBAC

Service Account:

```sh
$ kubectl create -f serviceaccount.yaml 
serviceaccount "nfs-client-provisioner" created
```
Create a Grant:
The default service account that’s associated with the provisioner namespace needs access to Kubernetes resources, so create a grant with Restricted Control permissions.

1. Navigate to the Grants page and click Create Grant.
2. In the left pane, click Resource Sets, and in the Type section, click Namespaces.
3. Enable the Apply grant to all existing and new namespaces option.
4. In the left pane, click Roles. In the Role dropdown, select Restricted Control.
5. In the left pane, click Subjects, and select Service Account.
6. In the Namespace dropdown, select nfs-client-provisioner, and in the Service Account dropdown., select default.
7. Click Create.

3. Finally, test your environment!

Now we'll test your NFS provisioner.

Deploy:

```sh
$ kubectl create -f test-claim.yaml -f test-pod.yaml
```

Now check your NFS Server for the file `SUCCESS`.

```sh
kubectl delete -f test-pod.yaml -f test-claim.yaml
```

Now check the folder renamed to `archived-???`.

4. Deploying your own PersistentVolumeClaim

To deploy your own PVC, make sure that you have the correct `storage-class` as indicated by your `class.yaml` file.

For example:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```