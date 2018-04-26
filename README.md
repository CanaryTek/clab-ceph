# CloudLab: Deploy a Ceph Cluster using SaltStack and DeepSea

In this CloudLab we deploy a Ceph cluster using openSUSE and DeepSea, a SaltStack formula to deploy Ceph clusters

At the end of Lab 1 we will have a fully functional Ceph cluster, managed by OpenAttic and with all services: Ganesha-NFS, iSCSI gateway, RadosGW, etc

## The Lab

This lab consists of 9 VM based on openSUSE Leap 42.3 with static IP

| Name | IP | RAM | Cores | Disc | Services | Description |
|------|----|-----|-------|------|----------|-------------| 
| ceph-deploy | 192.168.122.11 | 4GB | 4 | 1x40G | OpenATTIC, Grafana, salt-master | The host that will be used as deploy and monitoring host |
| ceph-test | 192.168.122.12 | 1GB | 1 | 1x40G | None | Just a test host to use as a client for iSCSI, NFS, etc |
| ceph-mon{1,2,3} | 192.168.122.2{1,2,3}) | 1GB | 1 | 1x40G | MON, MGR, MDS, iSCSI Gateway, NFS-Ganesha, RadosGW | Monitors and Service gateways |
| ceph-osd{1,2,3,4} | 192.168.122.3{1,2,3,4}) | 1GB | 1 | 1x40GB+2x30GB | OSD |  Ceph OSD storage hosts |

**NOTE:** As you can see, you will need a pretty powerfull computer to run all VM needed by this lab

## Common workflow

  0. Prepare the lab (download needed images)

```shell
sudo rake prepare_lab
```

  1. Edit the lab.yml file to adapt to your needs (al least add you SSH public key to be able to connect to the lab's VM)

  2. If using btrfs, create a "vm" subvolume so you can create snapshot at each Lab stage

```shell
sudo btrfs subvolume create vm
```

  3. Initialize the virtual machines for the lab

```shell
sudo rake init_vms
```

## Common tasks

  1. Start/Stop all VM belonging to the lab

```shell
sudo rake start_vms
sudo rake stop_vms
```

  2. Create snapshot at an important milestone (i.e. after installation)

```shell
sudo btrfs snap -r vm .snapshots/01_after_installation
```

  3. Revert to the previous snapshot

```shell
sudo btrfs subvol delete vm
sudo btrfs snap .snapshots/01_after_installation vm
```

## Labs

  * [Lab 1: Deploy_Ceph](labs/01_Deploy_Ceph.md)
  * [Lab 2: Pools](labs/02_Pools.md)
  * [Lab 3: RBD_Rados_Block_Device](labs/03_RBD_Rados_Block_Device.md)
  * [Lab 4: Data Placement](labs/04_Data_Placement.md)
  * [Lab 5: Crush Maps](labs/05_Crush_Maps.md)
  * [Lab 6: CephFS](labs/06_CephFS.md)
  * [Lab 7: Users and Access](labs/07_Users_and_Access.md)
  * [Lab 8: RadosGW](labs/08_RadosGW.md)

**TODO**

  * [Lab X: OSD failure]
  * [Lab X: MON failure]
  * [Lab X: Add new OSD host]
  * [Lab X: Delete a OSD host]
  * [Lab X: Replace a MON]
  * [Lab X: GaneshaNFS]
  * [Lab X: iSCSI]

