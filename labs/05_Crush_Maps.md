### CRUSH topology: add racks

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

        crushtool -c crushmap.bin -o crushmap.bin
        ceph osd setcrushmap -i crushmap.bin

 * This will cause data movement and cluster degradation. Monitor with "ceph -s"

