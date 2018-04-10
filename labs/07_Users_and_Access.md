# Users and access control

In this lab we will see how we can create custom users to allow access only to some services or pools

# User management

  * List users

        ceph auth list

  * Create pools test and test2

        ceph osd pool create test 128

  * Create a user that will be able to access only the "test" pool

        ceph auth get-or-create client.test mon 'allow r' osd 'allow rw pool=test' > ceph.client.test.keyring

  * This will dump the user auth to the ceph.client.test.keyring file

  * Copy the ceph.client.test.keyring file to /etc/ceph (or any other dir) in the ceph client host

## Access to pools using rados 

All the following commands should be run in the ceph client host

  * List pools

        rados lspools

  * It won't work because we need to specify the user and keyring to use (ceph defaults to admin user and checks some files in /etc/ceph)
  * Try again specifying user and keyring

        rados -k ceph.client.test.keyring --id test lspools

  * Since adding user and keyring options to all ceph commands can be a pain, we can setup argument in an environment variable

        export CEPH_ARGS="-k ceph.client.test.keyring --id test"
        rados lspools

  * List objects in test pool

        rados ls -p test

  * Put some objects in test pool

```
  rados put services /etc/services -p test
  rados put kk1 /etc/services -p test
  rados put kk2 /etc/services -p test
  rados put kk3 /etc/services -p test
```

  * Check thet object was created

        rados ls -p test

  * Try to list pool test2. You should get an error because test user only has permissions on pool test

        rados ls -p test22

### Access control with object_prefix

Since having a lot of pools can create a huge load on the ceph cluster, they can not be the only level for access control. We can setup access control based on the object name prefix

  * Create a user test2 with access to pool test, but only objects begining with "kk" 

        ceph auth get-or-create client.test2 mon "allow *" osd "allow rw pool=test object_prefix kk"

  * From the ceph client host, try to download objects from pool test. You should be able to get only the objects beginning with "kk"

        rados -k ceph.client.test2.keyring --id test2 -p test get kk1 kk1
        rados -k ceph.client.test2.keyring --id test2 -p test get kk2 kk2
        rados -k ceph.client.test2.keyring --id test2 -p test get services services

### Access control object_prefix

Since having a lot of pools can create a huge load on the ceph cluster, they can not be the only level for access control. We can create different namespaces in a pool, and allow access only to a given namespace

  * Modify test2 user to have access only to namespace nm1 in pool test

        ceph auth get-or-create client.test2 mon "allow *" osd "allow rw pool=test namespace=nm1"

  * With the admin user, put some objects in namespace nm1 in pool test

        rados put kk1 /etc/services -p test -N nm1
        rados put kk2 /etc/services -p test -N nm1

  * From the ceph client host, try to access the different namespaces in pool test

        rados -k ceph.client.test2.keyring --id test2 -p test -N nm1 ls
        rados -k ceph.client.test2.keyring --id test2 -p test ls

  * Only the command on namespace "nm1" should succeed

## Access control with rbd

  * As admin, create an rbd image

        rbd create testvm01-disk01 -s 2G

  * See the object prefix for the image

```
# rbd info testvm01-disk01
rbd image 'testvm01-disk01':
	size 2048 MB in 512 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.14a30643c9869
	format: 2
	features: layering
	flags: 
	create_timestamp: Sat Jan 27 20:26:32 2018

```

  * In this example, objects for that image will have a name starting with "rbd_data.149ef74b0dc51", we need to allos access to data and header

        ceph auth caps client.test mon "allow r" osd 'allow rx pool=rbd, allow rwx object_prefix rbd_header.14a30643c9869, allow rwx object_prefix rbd_data.14a30643c9869'

  * Now test user should be able to list images, but only mount "testvm01-disk01" 

  * Of course, we can also control access on a per pool basis, but as we said, many pools can create too much load on the ceph cluster

## Access control with CephFS

We will create several subdirectories in cephfs and allow the test user access to only one of them

  * Mount cephfs as the admin user to create subdirs (get the secret from the admin user keyring)

        mount -t ceph ceph-mon1:/ /mnt/ -oname=admin,secret=AQDBOGZai+YkAxAAEeUbkgyqCR1a4X9dZmNseQ==

  * Create several directories

        mkdir /mnt/test-home /mnt/private1 /mnt/private2

  * Allow the test user access to the cephfs_data pool and the subdir in mds. NOTE: with old kernels (before 4.x) you need to allow read access to the "/" path

        ceph auth caps client.test mon "allow r" osd 'allow * pool=cephfs_data' mds "allow * path=/test-home"

  * In recent versions you can also use the ceph fs authorize

        ceph fs authorize cephfs client.test / r /test-home rw

  * Mount the cephfs

        mount -t ceph ceph-mon1:/ /mnt/ -oname=test,secret=AQD0wmxawfKqFBAAhbRnIACauIOgYZ4nNAHAWg==

  * Check that you can only write in /mnt/test-home

  * You can also mount the subdirectory

        mount -t ceph ceph-mon1:/test-home /mnt/ -oname=test,secret=AQD0wmxawfKqFBAAhbRnIACauIOgYZ4nNAHAWg==

