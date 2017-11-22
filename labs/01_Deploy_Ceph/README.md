# Desploy Ceph cluster

## Cluster preparation

  * Add the following entries to /etc/hosts (change the IP prefix if needed)

```
192.168.122.11  ceph-deploy
192.168.122.12  ceph-test
192.168.122.21  ceph-mon1
192.168.122.22  ceph-mon2
192.168.122.23  ceph-mon3
192.168.122.31  ceph-osd1
192.168.122.32  ceph-osd2
192.168.122.33  ceph-osd3
192.168.122.34  ceph-osd4
```

  * Copy /etc/hosts to all nodes

  for h in ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; scp /etc/hosts root@$h:/etc/hosts; done

  * Disable ipv6 in all nodes (to avoid long delays)

```
echo > /etc/sysctl.d/disable-ipv6.conf <<EOF
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
```

  for h in ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; scp /etc/sysctl.d/disable-ipv6.conf root@$h:/etc/sysctl.d/disable-ipv6.conf; done
  reboot

  * Setup nameserver in all hosts

  echo "nameserver 192.168.122.1" > /etc/resolv.conf
  for h in ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'echo "nameserver 192.168.122.1" > /etc/resolv.conf'; done

  * Install and start NTP in all hosts

  for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'zypper --non-interactive in ntp'; done
  for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h "ntpdate pool.ntp.org; systemctl start ntpd; systemctl enable ntpd ntp-wait"; done

  * Install and start iptables in all hosts

  for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'zypper --non-interactive in iptables'; done

  * Update all nodes

  for h in ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'zypper --non-interactive update; reboot'; done
  zypper --non-interactive update; reboot

## Install DeepSea in deploy host

  zypper in salt-master-2016.11.4-8.1 salt-api-2016.11.4-8.1 salt-minion-2016.11.4-8.1

  * Install repo

  zypper ar "http://download.opensuse.org/repositories/filesystems:/ceph:/luminous/openSUSE_Leap_42.3/filesystems:ceph:luminous.repo"
  zypper refresh
  zypper install deepsea
  * Start salt-master

  systemctl enable salt-master.service ; systemctl start salt-master.service
  systemctl enable salt-api.service ; systemctl start salt-api.service

  * Install and start salt-minion in all hosts

  for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'zypper --non-interactive in salt-minion'; done
  for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'echo "master: ceph-deploy" > /etc/salt/minion.d/master.conf'; done
  for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'systemctl start salt-minion; systemctl enable salt-minion'; done

  * Accept all keys

  salt-key -A

  * Check salt is OK

```
  salt "*" test.ping
  salt "*" test.version
```

## Deploy Ceph with DeapSea

  * Setup ceph targets in /srv/pillar/ceph/deepsea_minions.sls

```
deepsea_minions: 'ceph*'
```

  * Run stage 0 (check and update)

  salt-run state.orch ceph.stage.0

  * Run stage 1 (discover)

  salt-run state.orch ceph.stage.1

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
role-igw/cluster/mon*.sls

# Rados GW
role-rgw/cluster/ceph-mon*.sls

# NFS
role-ganesha/cluster/ganesha*.sls

# openATTIC
role-openattic/cluster/openattic*.sls

# COMMON
config/stack/default/global.yml
config/stack/default/ceph/cluster.yml

## Profiles
profile-default/cluster/*.sls
profile-default/stack/default/ceph/minions/*.yml
```

  * We ran stage 2

  salt-run state.orch ceph.stage.2

  * Check pillar

```
  salt '*' pillar.items
```

  * Run stage 3 (Deploy)

  salt-run state.orch ceph.stage.3

  * Once the previous step completes, we should have a working ceph cluster. Check with

  ceph -s

  * Run stage 4 (Install additional services)

  salt-run state.orch ceph.stage.4

  * Check status

  ceph status

### Troubleshooting

#### NTP

```
ceph-deploy:~ # ceph health detail
HEALTH_WARN clock skew detected on mon.ceph-mon2, mon.ceph-mon3; Monitor clock skew detected 
mon.ceph-mon2 addr 192.168.122.22:6789/0 clock skew 0.269938s > max 0.05s (latency 0.00380513s)
mon.ceph-mon3 addr 192.168.122.23:6789/0 clock skew 1.03575s > max 0.05s (latency 0.00463017s)
```

##### Troubleshoot

  * Check salt-api

  curl -sSk http://localhost:8000/login -H 'Accept: application/x-yaml' -d eauth=sharedsecret -d username=admin -d sharedsecret=9944d99e-9601-461e-a454-c8eee2701847

### For salt-2017...

  * (If using salt 2017-...) There is an error in the salt-api service because it does nor connect to salt-master when running as user "salt". Change it so it runs as root

```
cat > /etc/systemd/system/salt-api.service <<EOF
[Unit]
Description=The Salt API
Documentation=man:salt-api(1) file:///usr/share/doc/salt/html/contents.html https://docs.saltstack.com/en/latest/contents.html
After=network.target

[Service]
Type=simple
LimitNOFILE=8192
ExecStart=/usr/bin/salt-api
TimeoutStopSec=3

[Install]
WantedBy=multi-user.target
EOF
```


