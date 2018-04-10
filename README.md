# CloudLab: Deploy a Ceph using SaltStack and DeapSea

In this CloudLab we deploy a Ceph cluster using openSUSE and DeapSea, a SaltStack formula to deploy Ceph clusters

## The Lab

This lab consists of 9 VM based on openSUSE Leap 42.3 with static IP

## Common workflow

  0. Prepare the lab (download needed images)

```
rake prepare_lab
```

  1. Edit the lab.yml file to adapt to your needs (if needed)

  2. If using btrfs, create a "vm" subvolume so you can create snapwhot at each Lab stage

```
btrfs subvolume create vm
```

  3. Initialize the virtual machines for the lab

```
rake init_vms
```

## Common tasks

  1. Start/Stop all VM belonging to the lab

```
rake start_vms
rake stop_vms
```

  2. Create snapshot at an important milestone (i.e. after installation)

```
btfs snap -r vm .snapshots/01_after_installation
```

  3. Revert to the previous snapshot

```
btrfs subvol delete vm
btfs snap .snapshots/01_after_installation vm
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

