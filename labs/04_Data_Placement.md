# Ceph data placement

We sill create a pool and an object and see how Ceph stores the object

  * Create a replicated pool with 128 pg

        ceph osd pool create data_test 128 128

  * Create an object with sample data

        echo "Ceph data test" > ceph-test.txt
        rados -p data_test put ceph-test ceph-test.txt

  * Check object

        rados -p data_test ls
        rados -p data_test get ceph-test -

  * See object mapping

        ceph-deploy:~ # ceph osd map data_test ceph-test
        osdmap e209 pool 'data_test' (8) object 'ceph-test' -> pg 8.e18db830 (8.30) -> up ([0,7,5], p0) acting ([0,7,5], p0)

  * In the previus example, the object is stored in PG 8.30 that is located in OSD 0, 7 and 5, and the primary OSD is 0

