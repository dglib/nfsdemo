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

By default Ubuntu exports NFS under NFS v2, v3 and v4, for testing against NFS v3 you can disable v4 by editing /etc/default/nfs-kernel-server

Reload the configuration blocking NFS v4/ re-enable to activate NFS Dynamic Volume provisioning.

    
    sudo systemctl restart nfs-config
    sudo systemctl restart nfs-kernel-server
    