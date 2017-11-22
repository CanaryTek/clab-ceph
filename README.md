# Ceph Lab

Lab to setup and manage a Ceph cluster using LabTools

## The Lab

This lab consists of 5 VM based on openSUSE Leap 42.3 with static IP

## Common workflow

  0. Prepare te lab

  ```rake :prepare_lab```

  1. First of all, create a brtrfs subvolume to store the VM's disks
  2. Define the VM (normally with empty disks or from template)
  3. We can create btrfs snapshots at any time, so we can rollback to a known state
  4. Start/Stop all VM belonging to a lab
  5. Each VM can be run in a different hypervisor
  6. We may need to define networks

