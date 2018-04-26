# Desploy Ceph cluster

## Cluster preparation

  * All these steps should be done on the ceph-deploy hosts. Connect using SSH

```shell
ssh root@192.168.122.11
```

  * Add the following entries to /etc/hosts (change the IP prefix if needed)

```shell
cat >> /etc/hosts <<EOF
192.168.122.11  ceph-deploy
192.168.122.12  ceph-test
192.168.122.21  ceph-mon1
192.168.122.22  ceph-mon2
192.168.122.23  ceph-mon3
192.168.122.31  ceph-osd1
192.168.122.32  ceph-osd2
192.168.122.33  ceph-osd3
192.168.122.34  ceph-osd4
EOF
```

  * Copy /etc/hosts to all nodes

```shell
for h in ceph-test ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; scp /etc/hosts root@$h:/etc/hosts; done
```

  * Disable ipv6 in all nodes (to avoid long delays)

```shell
cat > /etc/sysctl.d/disable-ipv6.conf <<EOF
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
for h in ceph-test ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; scp /etc/sysctl.d/disable-ipv6.conf root@$h:/etc/sysctl.d/disable-ipv6.conf; done
```

  * Setup nameserver in all hosts

```shell
echo "nameserver 192.168.122.1" > /etc/resolv.conf
for h in ceph-test ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'echo "nameserver 192.168.122.1" > /etc/resolv.conf'; done
```

  * Install and start NTP in all hosts

```shell
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'zypper --non-interactive in ntp'; done
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h "ntpdate pool.ntp.org; systemctl start ntpd; systemctl enable ntpd "; done
```

  * Install some dependencies in all hosts

```shell
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'zypper --non-interactive in iptables pciutils'; done
```

  * Update all nodes

```shell
for h in ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'zypper --non-interactive update; reboot'; done
zypper --non-interactive update; reboot
```

  * Install my openSUSE-Storage repo

```shell
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'zypper ar "https://download.opensuse.org/repositories/home:/kuko:/openSUSE-Storage/openSUSE_Leap_42.3/home:kuko:openSUSE-Storage.repo"'; done
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'zypper --non-interactive --gpg-auto-import-keys ref'; done
```

## Install DeepSea in deploy host

  * Install DeepSea (only on deploy host)

```shell
zypper in deepsea
```

  * Start salt-master

```shell
systemctl enable salt-master.service ; systemctl start salt-master.service
systemctl enable salt-api.service ; systemctl start salt-api.service
```

  * Install and start salt-minion in all hosts

```shell
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'zypper --non-interactive in salt-minion'; done
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'echo "master: ceph-deploy" > /etc/salt/minion.d/master.conf'; done
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'systemctl start salt-minion; systemctl enable salt-minion'; done
```

  * Accept all keys

```shell
salt-key -A
```

  * Check salt is OK

```shell
salt "*" test.ping
salt "*" test.version
```
  * Check salt-api (get the SHAREDSECRET from /etc/salt/master.d/sharedsecret.conf)

```shell
curl -sSk http://localhost:8000/login -H 'Accept: application/x-yaml' -d eauth=sharedsecret -d username=admin -d sharedsecret=SHAREDSECRET
```

## Deploy Ceph with DeepSea


  * Add deepsea grain to all ceph hosts

```shell
salt "ceph*" grains.append deepsea default
```

  * By default, DeepSea will update to latest kernels and automatically reboot nodes no stage 0. Since we prefer to reboot manually, we disable automatic reboots

```shell
cat > /srv/pillar/ceph/stack/global.yml <<EOF
stage_prep_master: default-update-no-reboot
stage_prep_minion: default-update-no-reboot
EOF
salt '*' saltutil.pillar_refresh
```

  * Run stage 0 (check and update)

```shell
salt-run state.orch ceph.stage.0
```

  * You may get some errors regarding kernel version checking. Don't worry, it's not critical (I will fix it ASAP)

  * Run stage 1 (discover)

```shell
salt-run state.orch ceph.stage.1
```

  * Setup policy  /srv/pillar/ceph/proposals/policy.cfg

```
## Cluster Assignment
cluster-ceph/cluster/ceph-*.sls

## Roles
# ADMIN
role-master/cluster/ceph-deploy*.sls
role-admin/cluster/ceph-deploy*.sls

# MON
role-mon/cluster/ceph-mon*.sls

# MGR (mgrs are usually colocated with mons)
role-mgr/cluster/ceph-mon*.sls

# MDS
role-mds/cluster/ceph-mon*.sls

# ISCSI GW
role-igw/cluster/ceph-mon*.sls

# Rados GW
role-rgw/cluster/ceph-mon*.sls

# NFS
role-ganesha/cluster/ceph-mon*.sls

# openATTIC
role-openattic/cluster/ceph-deploy.sls

# COMMON
config/stack/default/global.yml
config/stack/default/ceph/cluster.yml

## Profiles
profile-default/cluster/*.sls
profile-default/stack/default/ceph/minions/*.yml
```

  * You can also change network configuration in the following file

```
/srv/pillar/ceph/proposals/config/stack/default/ceph/cluster.yml
```

  * Run stage 2 (setup)

```shell
salt-run state.orch ceph.stage.2
```

  * Check pillar

```shell
salt '*' pillar.items
```

  * Run stage 3 (Deploy)

```shell
salt-run state.orch ceph.stage.3
```

  * Once the previous step completes, we should have a working ceph cluster. Check with

```shell
ceph -s
```

  * Run stage 4 (Install additional services) 

```shell
salt-run state.orch ceph.stage.4
```

  * Check status

```shell
ceph -s
```

### Complete stop/start

If you need to stop the whole cluster, follow this procedure to do a clean shutdown

#### Stop

  1. Stop the clients from using the RBD images/Rados Gateway on this cluster or any other clients.
  2. The cluster must be in healthy state before proceeding.
  3. Set the noout, norecover, norebalance, nobackfill, nodown and pause flags

```shell
ceph osd set noout
ceph osd set norecover
ceph osd set norebalance
ceph osd set nobackfill
ceph osd set nodown
ceph osd set pause
```

  4. Shutdown osd nodes one by one
  5. Shutdown monitor nodes one by one
  6. Shutdown admin node

#### Start

  1. Power on the admin node
  2. Power on the monitor nodes
  3. Power on the osd nodes
  4. Wait for all the nodes to come up , Verify all the services are up and the connectivity is fine between the nodes.
  5. Unset all the noout,norecover,noreblance, nobackfill, nodown and pause flags.

```shell
ceph osd unset noout
ceph osd unset norecover
ceph osd unset norebalance
ceph osd unset nobackfill
ceph osd unset nodown
ceph osd unset pause
```

  6. Check and verify the cluster is in healthy state, Verify all the clients are able to access the cluster.

