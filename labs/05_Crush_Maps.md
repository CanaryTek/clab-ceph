# CRUSH maps

CRUSH maps define how data placement is done

In this lab we will modify the CRUSH map to adapt it to different needs

## Distribute data across racks

  * View crush topology

        ceph osd tree

  * Add three racks

```
  ceph osd crush add-bucket rack01 rack 
  ceph osd crush add-bucket rack02 rack 
  ceph osd crush add-bucket rack03 rack 
```

  * View topology

        ceph osd tree

  * Move the new rack buckets below the root bucket

```
  ceph osd crush move rack01 root=default
  ceph osd crush move rack02 root=default
  ceph osd crush move rack03 root=default
```

 * Move hosts to corresponding racks. WARNING: This will cause data movement, so if you do it in a production cluster, do it step by step and wait until cluster is healthy before moving the next host. After each step run "ceph -s" and wait until cluster is healthy

```
  ceph osd crush move ceph-osd1 rack=rack01
  ceph osd crush move ceph-osd2 rack=rack02
  ceph osd crush move ceph-osd3 rack=rack03
  ceph osd crush move ceph-osd4 rack=rack03
```

Now we need to tell Ceph to enforce distribution of replicas among racks

 * Get the crushmaps

        ceph osd getcrushmap -o crushmap.bin

 * Convert the crushmap to a human readable txt

        crushtool -d crushmap.bin -o crushmap.txt

 * To distribute objects at the rack level instead of host level, change this line

        -step chooseleaf firstn 0 type host
        +step chooseleaf firstn 0 type rack

 * And apply the change

        crushtool -c crushmap.txt -o crushmap.bin
        ceph osd setcrushmap -i crushmap.bin

 * This will cause data movement and cluster degradation. Monitor with "ceph -s"

## Define a map for SSD disks

If we have rotation and SSD disks in the same cluster, we need to define a different map for the SSD disks, and create pools that will be stored on the SSD disks

  * Set noup flag to keep the osd down until everything is ready

        ceph osd set noup

  * Setup the disks in all hosts

  ceph-disk prepare --bluestore /dev/vdd

  * In most cases in a real cluster, the type of disks would be detected and used as a "device-class". You can also specify a device-class in the previuos command adding ``--device-class ssd``

  * We can also change the device-class with the following commands (for osd 8, 9, 10 and 11)

        $ ceph osd crush rm-device-class osd.8 osd.9 osd.10 osd.11
        done removing class of osd(s): 8,9,10,11
        $ ceph osd crush set-device-class ssd osd.8 osd.9 osd.10 osd.11
        set osd(s) 8,9,10,11 to class 'ssd'

  * New we need to modify any existing placemant group to use only the "hdd" device-class, to avoid existing data to be "redistributed" to the new SSD disks

  * Use the method used in the previus example to dump que crushmap

```
ceph osd getcrushmap -o crushmap.bin                                                                                                    
crushtool -d crushmap.bin -o crushmap.txt                                                                                               
```

  * Edit crushmap.txt and edit the default rule (id 0) to choose only from osd with device-class hdd

```
  -step take default
  +step take default class hdd
```

  * "Compile" and apply crushmap

```
crushtool -c crushmap.txt -o crushmap.bin
ceph osd setcrushmap -i crushmap.bin
```

  * When everything is ready, remove the "noup" flag

        ceph osd unset nohup

  * If everything worked as expected, you shouldn't see any data movement

  * Now you need to create a new placement rule to choose only from SSD disks

        ceph osd crush rule create-replicated ssd-disks default host ssd

  * If you use per rack distribution, use "rack" instead of "host"

        ceph osd crush rule create-replicated ssd-disks default rack ssd

  * Now create a pool and assign the new placement rule and all data on that pool will be stored on the fast SSD disks

        ceph osd pool create fast-pool 128 128 replicated ssd-disks

  * Check that the new pool has a different crush_rule

        ceph osd pool ls detail

  * Now we can check data placement. Create a new object on the new pool

        echo TEST | rados put object1 - -p fast-pool

  * Check object was created

        rados ls -p fast-pool

  * Check object placement (it should be located only on the new SSD osd we just added)

        $ ceph osd map fast-pool TEST
        osdmap e691 pool 'fast-pool' (11) object 'TEST' -> pg 11.ed855b7a (11.7a) -> up ([8,9,11], p8) acting ([8,9,11], p8)

