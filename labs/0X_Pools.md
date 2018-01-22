# Working with pools

## List and create

  * List pools

  ceph osd pool ls [detail]

  * Create a pool with 128 placement groups

  ceph osd pool create test 128

## Snapshots

  * Put some content

  rados -p test put services /etc/services

  * List objects

  rados ls -p test

  * Create a snapshot

  rados mksnap test-snap01 -p test

  * View snapshots

  rados lssnap -p test

  * Remove the object

  rados rm services -p test

  * List objects

  rados ls -p test

  * We still can see the object because it's present in the snapshot. But and ls for the object should be empty

  rados ls services -p test

  * View object in snapshots (overlap should be zero)

  rados -p test listsnaps services

  * Rollback snapshot

  rados rollback services test-snap01 -p test 

## Delete a pool

  * For safety, pool deletion in disabled by default. You need to enable it in the MONs

  ceph tell mon.* injectargs '--mon_allow_pool_delete=1'

  * For safety (again) to delete a pool you need to give the pool name twice and a "special" option

  ceph osd pool delete test test --yes-i-really-really-mean-it

  * Once the pool is deleted, disable pool deletion again

  ceph tell mon.* injectargs '--mon_allow_pool_delete=0'

