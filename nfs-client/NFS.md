# My NFS server configuration for Testing
## Starting with Ubuntu Server 16.04
1. Add nfs-server
    ```
    sudo apt-get install -y nfs-server
    ```
2. Make the nfs volume, set permissions
    ```
    sudo mkdir -p /nfs/vol1
    sudo chmod 755 /nfs/vol1
    sudo chown nobody:nogroup /nfs/vol1
    ```
3. Create the export and present the filesystem
    ```
    sudo echo "/nfs/vol1  *(rw,sync,no_subtree_check,all_squash,anonuid=65534,anongid=65534)" >> /etc/exports
    sudo exportfs -af
    ```
The volume "/nfs/vol1" will be the local filesystem exported as a share
I'm using a wildcard here "*" to allow all remote mounts. Privileges are "rw" = read-write, "sync" = commit writes to disk before client acknowledgement, "no_subtree_check" = we're going to force permissions on writes, so I'll ignore these filesystem checks to improve performance, "all_sqash" = all uid's and gid's (including root) will be squashed (converted) on the nfs server side, "anonuid/anongid=65534" = this forces all users to write to the filesystem on the nfs server share as user 65534 (my uid/gid = nobody:nogroup).

By default Ubuntu exports NFS under NFS v2, v3 and v4. For testing against NFS v3 ONLY, you can disable NFS v4 by editing /etc/default/nfs-kernel-server

 
```
Change line RPCNFSDCOUNT="8"
To RPCNFSDCOUNT="8 --no-nfs-version 4"

Change line RPCNFSDOPTS=""
To RPCNFSDOPTS=" --no-nfs-version 4"
```

Reload the configuration blocking NFS v4

    
    sudo systemctl restart nfs-config
    sudo systemctl restart nfs-kernel-server
    