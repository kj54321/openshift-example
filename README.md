# How to create wordpress apps on Openshift Paas

The initiative of this guide is to create a rough illustration for full stack (LB + Frontend + DB) pods on Openshift. Before you proceed, it is assumable that you have configed router and storage in the platfrom. In particular, Persistent Volumes with NFS refer to [OpenShift persistent storage guide](https://docs.openshift.com/enterprise/3.2/dev_guide/persistent_volumes.html), which explains how to use these Persistent Volumes as data storage for applications.

## NFS Provisioning

We'll be creating NFS exports on the local machine.  I here take an example from [offical docs](https://access.redhat.com/documentation/en/openshift-enterprise/3.0/paged/administrator-guide/chapter-15-persistent-storage-using-nfs) under el7 system.  The provisioning process may be slightly different based on linux distribution or the type of NFS server being used.

Create two NFS exports, each of which will become a Persistent Volume in the cluster.

```
# the directories in this example can grow unbounded
# use disk partitions of specific sizes to enforce storage quotas
mkdir /OSE_wordpress /OSE_mysql

# Set appropriate privilege and security, export will also be restricted 
# to the same UID/GID that wrote the data
chown nfsnobody:nfsnobody /OSE_wordpress /OSE_mysql
chmod 700 /OSE_wordpress /OSE_mysql

# Add to /etc/exports
/OSE_wordpress 192.168.0.*(rw,async,all_squash)
/OSE_mysql 192.168.0.*(rw,async,all_squash)

# Enable the new exports without bouncing the NFS service
exportfs -a

# Verify whether it  takes effect
systemctl status nfs-server
showmount -e localhost
```

## Security

### SELinux

By default, SELinux does not allow writing from a pod to a remote NFS server. The NFS volume mounts correctly, but is read-only.

To enable writing in SELinux on each node:

```
# -P makes the bool persistent between reboots.
$ setsebool -P virt_use_nfs 1
```

### IPtables

We can used fixed ports for nfs / rpc bind to ease the firewall settings. Also some kernel tuning for better performance.

```
sed -i '
s/RPCMOUNTDOPTS=""/RPCMOUNTDOPTS="-p 20080"/
s/STATDARG=""/STATDARG="-p 50080"/
' /etc/sysconfig/nfs

sed -i '/COMMIT/ i \
# BEGIN NFS server \
-A INPUT -p tcp -m state --state NEW -m tcp --dport 53248 -j ACCEPT \
-A INPUT -p tcp -m state --state NEW -m tcp --dport 50080 -j ACCEPT \
-A INPUT -p tcp -m state --state NEW -m tcp --dport 20080 -j ACCEPT \
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2049 -j ACCEPT \
-A INPUT -p tcp -m state --state NEW -m tcp --dport 111 -j ACCEPT \
# END NFS server' /etc/sysconfig/iptables

echo '
fs.nfs.nlm_tcpport=53248
fs.nfs.nlm_udpport=53248
' >> /etc/sysctl.conf

``` 

## NFS Persistent Volumes

### PV

Each NFS export becomes its own Persistent Volume in the cluster, called pv in this phase.

```
# Create the persistent volumes for NFS.
$ oc create -f pv-mysql.yaml
$ oc create -f pv-wordpress.yaml 
$ oc get pv
NAME              LABELS                             CAPACITY      ACCESSMODES   STATUS      CLAIM                      REASON
mysql             <none>                             5368709120    RWO           Available                              
wordpress         <none>                             1073741824    RWO,RWX       Available 
```

### PV claim (PVC)

Claim to allocate space for apps and define [accessModes](http://kubernetes.io/docs/user-guide/persistent-volumes/#access-modes-1).

```
$ oc create -f pvc-mysql.yaml
$ oc create -f pvc-wp.yaml
$ oc get pvc
NAME          LABELS    STATUS    VOLUME
claim-mysql   map[]     Bound     mysql
claim-wp      map[]     Bound     wordpress

```

Now the volumes are ready to be used by applications in the cluster.

## Deploy Apps (Wordpress)

Please notice that Openshift use deploymentConfig, thanks to [upstream examples](https://github.com/kubernetes/kubernetes/tree/master/examples/mysql-wordpress-pd), it is slightly different than kubernetes.

```
# deploy pods
$ oc create -f pod-mysql.yaml
# Comment imagePullPolicy in file if pulled from remote
$ oc create -f pod-wordpress.yaml

# deploy service
$ oc create -f service-mysql.yaml
$ oc create -f service-wp.yaml

# expose to external network
$ oc expose service wpfrontend --hostname=strangelove.farm.cloudapps.example.com
# or
$ oc edit route/wpfrontend

```

Then we can visit the website via given hostname provided that DNS has been properly configed.