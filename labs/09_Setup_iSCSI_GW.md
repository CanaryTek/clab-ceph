# Setup a Ceph iSCSI GW

In this lab we assume you already completed Lab 1 and you have a working Ceph cluster

## Create a iSCSI target

  * Login to openattic Web configuration
  * Go to "iSCSI" config
  * Create a new target
    * Add all available GW as portals
    * Add or create a RBD image

## iSCSI client

Now we will configure the ceph-test host as a iscsi initiatior to test multipath and failover

  * Connect to the ceph-test vm (192.168.122.12)

```shell
ssh root@192.168.122.12
```

  * Install needed software 

```shell
zypper in yast2-iscsi-client yast2-multipath yast2-storage
```
 
  * Run Yast
    * Enable multipath
    * iSCSI Initiatior
    * Go to "Discovered Targets"
    * Enter IP of one of the targets. It will discover all portals for the target
    * Login to all of them



  * Setup multipath for ALUA failover


  * Restart multipathd

```shell
systemctl restart multipathd
```

```shell
360014054dfef910bfd8306da0fd4980d dm-0 SUSE,RBD
size=1.0G features='2 queue_if_no_path retain_attached_hw_handler' hwhandler='1 alua' wp=rw
`-+- policy='service-time 0' prio=50 status=active
  |- 3:0:0:0 sda 8:0  active ready running
  |- 4:0:0:0 sdb 8:16 active ready running
  `- 2:0:0:0 sdc 8:32 active ready running
```

  * Create a XFS filesystem and mount it

```shell
mkfs  /dev/mapper/360014054dfef910bfd8306da0fd4980d
mount /dev/mapper/360014054dfef910bfd8306da0fd4980d /mnt
```

  * Create a test file

```shell
echo "TEST1" > /mnt/__Test_file.txt
cat /mnt/__Test_file.txt
```

### Failover test

  * Make sure you have the tree paths enabled

```shell
ceph-test:~ # multipath -l
360014054dfef910bfd8306da0fd4980d dm-0 SUSE,RBD
size=1.0G features='2 queue_if_no_path retain_attached_hw_handler' hwhandler='1 alua' wp=rw
`-+- policy='service-time 0' prio=0 status=active
  |- 3:0:0:0 sda 8:0  active undef unknown
  |- 4:0:0:0 sdb 8:16 active undef unknown
  `- 2:0:0:0 sdc 8:32 active undef unknown
```

  * Now kill one of the gateways


```shell
sudo virsh destroy clab.ceph-mon3
```

  * You should see one of the paths as "failed", but everything should be working

```shell
ceph-test:~ # multipath -l
360014054dfef910bfd8306da0fd4980d dm-0 SUSE,RBD
size=1.0G features='2 queue_if_no_path retain_attached_hw_handler' hwhandler='1 alua' wp=rw
`-+- policy='service-time 0' prio=0 status=active
  |- 3:0:0:0 sda 8:0  failed undef unknown
  |- 4:0:0:0 sdb 8:16 active undef unknown
  `- 2:0:0:0 sdc 8:32 active undef unknown

```

  * Make sure yo can still read and write to the mounted volume

```shell                                                                                                                                                                    
echo "TEST1" > /mnt/__Test_file.txt                                                                                                                                         
cat /mnt/__Test_file.txt                                                                                                                                                    
```                                                                                                                                                                         

  * Now start the failed gateway

```shell
sudo virsh destroy clab.ceph-mon3
```

  * After some time, all paths should be active again

```shell
ceph-test:~ # multipath -l
360014054dfef910bfd8306da0fd4980d dm-0 SUSE,RBD
size=1.0G features='2 queue_if_no_path retain_attached_hw_handler' hwhandler='1 alua' wp=rw
`-+- policy='service-time 0' prio=0 status=active
  |- 3:0:0:0 sda 8:0  active undef unknown
  |- 4:0:0:0 sdb 8:16 active undef unknown
  `- 2:0:0:0 sdc 8:32 active undef unknown
```

