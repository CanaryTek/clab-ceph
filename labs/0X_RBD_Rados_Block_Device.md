# RBD: Rados Block Device

## Create a new rbd

  * In the recent Ceph releases, the default pool for RBD (rbd) is not created on install, so we need to create it

  ceph osd pool create rbd 128 128

  * For all commands we can specify en alternate pool with the "-p" parameter

  * Create the RBD

  rbd create vm01-disk0 --size=1G

  * You can list the objects created on the pool. Youo will se several control objects

  rados ls -p rbd

  * List RBDs

  rbd list

  * Get extended info

  rbd info vm01-disk0

## Use the RBD

  * Map the RBD to a device

  rbd map vm01-disk0

  * Depending on Ceph and kernel version you may get an arror about features not supported by the kernel. This may happen because the RBD kernel driver is maintained by the kernel, and may be behind the installed Ceph version. If that happens, you will need to disable the conflicting features in the rbd pool

  rbd feature disable RBD CONFLICTING_FEATURE

  * List all mapped RBD

  rbd showmapped

  * Create a filesystem and mount

  mkfs -t ext4 /dev/rbd0
  mount /dev/rbd0 /mnt/
  df

  * See disk usage

  rbd du

  * Copy some content and see hoy usage increases

  dd if=/dev/zero of=/mnt/TEST.dd bs=1M count=200
  df
  rbd du

  * See how RBD split data in many RADOS objects on the rbd pool

  rados ls -p rbd

  * You will see many rbd_data.ID.SEGMENT objects. The ID identifies the RBD (it's the RBD "block_name_prefix" attribute, you can see it with "rbd info RBD")

## Resizing RBD

  * Reaize

  rbd resize vm01-disk0 --size=2G

  * Check that the kernel see the new size

  lsblk

  * Expand filesystem

  resize2fs /dev/rbd0
  df

## Using snapshots

  * Create a snapshot

  rbd snap create rbd/vm01-disk0@snap1

  * Delete the test file

  rm /mnt/TEST.dd

  * List snapshots

  rbd snap ls rbd/vm01-disk0

  * Rollback (you need to unmount first)

  umount /mnt
  rbd rollback rbd/vm01-disk0@snap1 

  * Mount and check the file is back

  mount /dev/rbd0 /mnt
  ls /mnt

  * Delete a snapshot

  rbd snap rm rbd/vm01-disk0@snap1

  * Delete ALL snapshots of a RBD

  rbd snap purge rbd/vm01-disk0

## RBD clones

We can create a RW clone from a snapshot. That clones can have their own snapshots and clones

  * Create a new RBD and a clone

  rbd create os-base --size 2G
  rbd map os-base
  mkfs -t ext4 /dev/rbd0
  mount /dev/rbd0 /mnt/
  echo "THIS IS THE BASE OS" > /mnt/BASE_OS.txt
  umount /mnt
  rbd unmap os-base
  rbd snap create rbd/os-base@snap1

  * To use a snaphot as parent for clones, you need to protect it from accidente deletion

  rbd snap protect rbd/os-base@snap1

  * Create clone

  rbd clone rbd/os-base@snap1 vm02-disk0

  * If you run "rbd info" on the clone, you will see it's parent clone

```
ceph-test:~ # rbd info vm02-disk0
rbd image 'vm02-disk0':
	size 2048 MB in 512 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.621c238e1f29
	format: 2
	features: layering
	flags: 
	create_timestamp: Sat Jan 20 13:29:30 2018
	parent: rbd/os-base@snap1
	overlap: 2048 MB
```

  * You can "flatten" a clone, wich means ceph will copy all segments from it's parent snapshot, so it exists as an independent RBD and you can delete the original parent clone

  rbd flatten vm02-disk0
  rbd info vm02-disk0 

  * Now, "rbd info" should not report any "parent"

  * Now we can delete the original snapshot (unprotecting it first)

  rbd snap unprotect rbd/os-base@snap1
  rbd snap rm rbd/os-base@snap1


