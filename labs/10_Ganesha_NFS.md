# Setup an NFS gateway with GaneshaNFS

In this lab we will setup and export the CephFS filesystem via NFS

We will also setup and test a keepalived based HA setup

## Prepare a filesystem to export

We will create a "test" directory under the CephFS root to use in our tests

  * Mount the CephFS in the deploy host (read ADMIN_KEY from /etc/ceph/ceph.client.admin.keyring)

```shell
mount -t ceph 192.168.122.21,192.168.122.22,192.168.122.23:/ /mnt -oname=admin,secret=ADMIN_KEY
```

  * Now you have the CephFS root mounted under /mnt
  * Create a test directory

```shell
mkdir /mnt/test
```

  * Give everybody write permissions (because we have root_squash enabled by default)

```shell
chmod 777 /mnt/test
```

## Mount on client

  * Install needed software

```shell
zypper in nfs-utils
```

  * Mount the CephFS via NFS

```shell
mount -t nfs 192.168.122.21:/cephfs /mnt/
```

  * Create a test file

```shell
echo 1 > /mnt/test/_uno.txt
```

  * Check that you see the created file also from de CephFS mounted on the deploy host

Ok, that was easy. But we are mounting from one of our 3 gateways. That means if that particular gateway fails, our NFS mount will fail. We have a Single Point Of Failure (SPOF) wich is something we need to avoid

In the next section we will correct this problem

## HA NFS with keepalived 

Since we are exporting the same filesystems in all three GaneshaNFS gateways, all we need to do is use some software that allows us to manage a "floating IP" that is assigned to a "master node" and assign the "floating IP" to another node if the "master node" fails. This is exactly what keepalived does.

We are going to create a keepalived cluster with a "floating IP" for the NFS service. The client will use this IP to mount the NFS share, this way, if the main node fails, keepalived will setup the service IP in a different node and the client will recover the NFS share in the new active node (remember that with Ganesha NFS **all** nodes can manage the CephFS filesystem)

  * Install keepalived on all NFS GW hosts

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h 'zypper --non-interactive in keepalived'; done
```

  * Create a basic keepalived config (change the virtual IP if needed )

```shell
cat > keepalived.conf <<_EOF_
global_defs {
   router_id GANESHA-NFS-HA
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.122.20
    }
}
_EOF_
```

  * Copy the keepalived config file to all NFS GW nodes

```shell
for h in ceph-mon{1,2,3}; do echo $h; scp keepalived.conf $h:/etc/keepalived ; done
```

  * Start and enable keepalived service in all NFS GW nodes

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h systemctl start keepalived; ssh $h systemctl enable keepalived; done
```

  * Make sure the virtual IP is active in one of the hosts

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h ip a; done
```

### Failover test

Now we are going to mount the NFS share using the previous floating IP and force a failure on the main node
The floating IP should failover to a different node and, after some time, the NFS should recover the NFS connection

  * Mount the CephFS via NFS (using the floating IP)

```shell
mount -t nfs 192.168.122.20:/cephfs /mnt/
```

  * Run this test command

```shell
while true; do sleep 1; date | tee -a /mnt/test/_test.txt ; done
```

  * Kill the VM with the floating IP

  * The command should halt and resume after some time
