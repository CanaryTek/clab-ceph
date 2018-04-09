# CephFS

## Mounting

  * Mount a CephFS from the MON hosts (you need to sepecify the client name and the secret from the keyring)

  mount -t ceph 192.168.122.21,192.168.122.22,192.168.122.23:/ /mnt -oname=admin,secret=AQAaeFxaAAAAABAAFQK/pP6+QHQqc2ATztlx7A==

  * You can also mount a different filesystem with (i.e. fs2)

  mount -t ceph 192.168.122.21,192.168.122.22,192.168.122.23:/ /mnt -oname=admin,secret=AQAaeFxaAAAAABAAFQK/pP6+QHQqc2ATztlx7A==,mds_namespace=fs2

  * NOTE: multiple CephFS filesystems is still considered an experimental feature

## Snapshots

Snapshots are handled with simple directory semantics (ls, mkdir, rmdir, etc) in a hidden ".snap" directory

  * Snapshot is still (as of luminous) considered an experimental feature, so we need to enable them explicitly

  ceph fs set cephfs allow_new_snaps true --yes-i-really-mean-it

  * Create a snapshot named 20180409

  mkdir /mnt/.snap/20180409

  * List snapshots

  ls /mnt/.snap

  * View contents of a snapshot

  ls /mnt/.snap/20180409

  * Delete a snapshot

  rmdir /mnt/.snap/20180409
