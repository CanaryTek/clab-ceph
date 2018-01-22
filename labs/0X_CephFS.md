# CephFS

  * Mount a CephFS from the MON hosts (you need to sepecify the client name and the secret from the keyring)

  mount -t ceph 192.168.122.21,192.168.122.22,192.168.122.23:/ /mnt -oname=admin,secret=AQAaeFxaAAAAABAAFQK/pP6+QHQqc2ATztlx7A==

  * You can also mount a different filesystem with (i.e. fs2)

  mount -t ceph 192.168.122.21,192.168.122.22,192.168.122.23:/ /mnt -oname=admin,secret=AQAaeFxaAAAAABAAFQK/pP6+QHQqc2ATztlx7A==,mds_namespace=fs2

  * NOTE: multiple CephFS filesystems is still considered an experimental feature
