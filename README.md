# Ty Case Study

## Overview
This case study demonstrates the process of adding and mounting a new disk (`/dev/sdb`) in an Ubuntu system.

## Steps

### 1. Update System Packages
```sh
sudo apt-get update -y
```

### 2. Check Available Disks
```sh
lsblk
```
**Output:**
```
sdb                         8:16   0    1T  0 disk
```

### 3. Partition the Disk
```sh
sudo fdisk /dev/sdb
```
- Create a new primary partition (`p`)
- Accept default first and last sectors
- Write changes (`w`)

### 4. Verify the New Partition
```sh
lsblk
```
**Output:**
```
sdb                         8:16   0    1T  0 disk
└─sdb1                      8:17   0 1024G  0 part
```

### 5. Format the Partition
```sh
sudo mkfs.ext4 /dev/sdb1
```

### 6. Create a Mount Point
```sh
sudo mkdir /ceph_disk
```

### 7. Mount the Partition
```sh
sudo mount -t ext4 /dev/sdb1 /ceph_disk
```

### 8. Get the UUID of the Partition
```sh
ls -lah /dev/disk/by-uuid/ | grep sdb1
```
**Output:**
```
lrwxrwxrwx 1 root root  10 Mar 21 19:03 7c7e1529-0eec-4161-a99e-1fecf4d88d8c -> ../../sdb1
```

### 9. Add the Partition to fstab for Persistent Mounting
Edit the `/etc/fstab` file using `vi` or another text editor:
```sh
sudo vi /etc/fstab
```
Add the following line at the end of the file:
```
/dev/disk/by-uuid/7c7e1529-0eec-4161-a99e-1fecf4d88d8c /ceph_disk ext4 defaults 0 2
```

### 10. Verify Mounting on Reboot
```sh
lsblk
```

## Conclusion
This process ensures that the additional disk is properly partitioned, formatted, and persistently mounted for future use.

# Setting Up Loopback Devices for Ceph

## Step 1: Create Loopback Disk Images
```bash
sudo truncate -s 100G /ceph_disks/loop101.img

for i in {101..110}; do
  sudo truncate -s 100G /ceph_disk/loop$i.img
done
```

Verify the created files:
```bash
ls /ceph_disk/
```
Expected output:
```bash
loop101.img  loop102.img  loop103.img  loop104.img  loop105.img  loop106.img  loop107.img  loop108.img  loop109.img  loop110.img
```

## Step 2: Create Loopback Devices
Check available loop devices:
```bash
grep loop /proc/devices
```
Output should show:
```bash
7 loop
```

Create the loop devices:
```bash
for i in {101..110}; do
  sudo mknod /dev/loop$i b 7 $i
done
```

Verify the devices:
```bash
ls -lah /dev/loop1*
```
Expected output:
```bash
brw-r--r-- 1 root root 7, 101 Mar 21 19:18 /dev/loop101
brw-r--r-- 1 root root 7, 102 Mar 21 19:18 /dev/loop102
...
brw-r--r-- 1 root root 7, 110 Mar 21 19:18 /dev/loop110
```

## Step 3: Create a Systemd Service for Loopback Disks

```bash
cat <<EOT | sudo tee -a /etc/systemd/system/loopbackdisk.service

[Unit]
Description=loopback devices disk
DefaultDependencies=no
Conflicts=umount.target
After=local-fs.target systemd-udevd.service
Requires=systemd-udevd.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'for i in {101..110}; do /sbin/losetup /dev/loop$i /ceph_disk/loop$i.img; done'
ExecStop=/bin/bash -c 'for i in {101..110}; do /sbin/losetup -d /dev/loop$i; done'
TimeoutSec=60
RemainAfterExit=yes

[Install]
WantedBy=local-fs.target
EOT
```

Reload systemd and enable the service:
```bash
systemctl daemon-reload
systemctl restart loopbackdisk.service
systemctl enable loopbackdisk.service
systemctl status loopbackdisk.service
```

Check loopback devices:
```bash
lsblk
```
Expected output:
```bash
loop101                     7:101  0  100G  0 loop
loop102                     7:102  0  100G  0 loop
...
loop110                     7:110  0  100G  0 loop
```

# Installing Cephadm

```bash
sudo apt install -y cephadm
```

Expected installation output:
```bash
The following NEW packages will be installed:
  bridge-utils cephadm containerd dns-root-data dnsmasq-base docker.io pigz runc ubuntu-fan
0 upgraded, 9 newly installed, 0 to remove and 119 not upgraded.
Need to get 78.8 MB of archives.
After this operation, 301 MB of additional disk space will be used.

# Ceph Disk Setup and Installation (Ceph Reef)

### Install cephadm
```bash
sudo apt install -y cephadm
```

This also installs dependencies such as Docker, containerd, etc.

### Add Ceph Reef Repository
```bash
sudo cephadm add-repo --release reef
```

### (Optional) Remove cephadm and Install Specific Version via RPM
In case a specific version of `cephadm` is needed (e.g., 18.2.4), it can be downloaded manually. This is sometimes required if the default version in Ubuntu's repo is outdated.

```bash
sudo apt remove --purge cephadm -y
curl --silent --remote-name --location https://download.ceph.com/rpm-18.2.4/el9/noarch/cephadm
chmod +x cephadm
mv cephadm /usr/sbin/
```

> 📚 Reference: [Ceph Official Downloads](https://docs.ceph.com/en/latest/install/)

###  Verify Installation
```bash
which cephadm
cephadm version
```
Output:
```plaintext
/usr/sbin/cephadm
cephadm version 18.2.4 (...) reef (stable)
```

##  Bootstrap the Ceph Cluster

### Prepare SSH Keys (already available in `/root/.ssh/`)
- `/root/.ssh/cephadm_rsa`
- `/root/.ssh/cephadm_rsa.pub`

### Step 2: Run Bootstrap
```bash
sudo cephadm bootstrap \
  --mon-ip 172.25.0.91 \
  --initial-dashboard-user admin \
  --initial-dashboard-password admin \
  --dashboard-password-noupdate \
  --ssh-private-key=/root/.ssh/cephadm_rsa \
  --ssh-public-key=/root/.ssh/cephadm_rsa.pub
```

### Output Summary
- Mon and Mgr services deployed
- Ceph dashboard enabled
- Default services deployed (Prometheus, Grafana, Alertmanager, etc.)

Dashboard Access:
```
URL: https://ubuntu01:8443/
User: admin
Password: admin
```

### Enter Ceph Shell
```bash
cephadm shell
```
> This gives access to Ceph CLI inside the container context.

Ready for:
- OSD provisioning with loopback disks
- Pool configuration (replication, EC)
- Squid upgrade for mon/mgr (in next steps)

# Ceph Cluster Setup and Upgrade

## Overview

This document outlines the full lifecycle of a Ceph cluster setup using Cephadm with loopback OSDs, including:

- Host configuration and key setup
- OSD provisioning with loopback devices
- Pool creation (replicated + erasure coded)
- Application enablement
- Version validation
- Selective upgrade of Mon and Mgr daemons to Squid (v19.x)

---

## Host Configuration & SSH Setup

### Copy SSH Public Key to Hosts
```bash
ssh-copy-id -f -i /root/.ssh/cephadm_rsa.pub trendyol@172.25.0.92
ssh-copy-id -f -i /root/.ssh/cephadm_rsa.pub trendyol@172.25.0.93
```

### Authorize Root on Each Host
```bash
cp -p /home/trendyol/.ssh/authorized_keys /root/.ssh/authorized_keys
chown root:root /root/.ssh/authorized_keys
```

### Install Docker on ubuntu02 and ubuntu03
```bash
sudo apt install -y docker.io
```

---

## Adding Hosts to Ceph Cluster

```bash
ceph orch host add ubuntu02 172.25.0.92 --labels=mon,mgr
ceph orch host add ubuntu03 172.25.0.93 --labels=mon,mgr
```

```bash
ceph orch host ls
```

Expected Output:
```plaintext
HOST      ADDR         LABELS   STATUS
ubuntu01  172.25.0.91  _admin
ubuntu02  172.25.0.92  mon,mgr
ubuntu03  172.25.0.93  mon,mgr
```

---

## Adding OSDs via Loop Devices

Assuming loop devices `/dev/loop101` to `/dev/loop110` are prepared on each host:

```bash
for i in {101..110}; do
  sudo ceph orch daemon add osd --method raw ubuntu01:/dev/loop$i
  sudo ceph orch daemon add osd --method raw ubuntu02:/dev/loop$i
  sudo ceph orch daemon add osd --method raw ubuntu03:/dev/loop$i
done
```

---

### OSD Tree
```bash
ceph osd tree
```
ID  CLASS  WEIGHT   TYPE NAME          STATUS  REWEIGHT  PRI-AFF
-1         2.93060  root default
-3         0.97687      host ubuntu01
 0    hdd  0.09769          osd.0          up   1.00000  1.00000
 1    hdd  0.09769          osd.1          up   1.00000  1.00000
 2    hdd  0.09769          osd.2          up   1.00000  1.00000
 3    hdd  0.09769          osd.3          up   1.00000  1.00000
 4    hdd  0.09769          osd.4          up   1.00000  1.00000
 5    hdd  0.09769          osd.5          up   1.00000  1.00000
 6    hdd  0.09769          osd.6          up   1.00000  1.00000
 7    hdd  0.09769          osd.7          up   1.00000  1.00000
 8    hdd  0.09769          osd.8          up   1.00000  1.00000
 9    hdd  0.09769          osd.9          up   1.00000  1.00000
-5         0.97687      host ubuntu02
10    hdd  0.09769          osd.10         up   1.00000  1.00000
11    hdd  0.09769          osd.11         up   1.00000  1.00000
12    hdd  0.09769          osd.12         up   1.00000  1.00000
13    hdd  0.09769          osd.13         up   1.00000  1.00000
14    hdd  0.09769          osd.14         up   1.00000  1.00000
15    hdd  0.09769          osd.15         up   1.00000  1.00000
16    hdd  0.09769          osd.16         up   1.00000  1.00000
17    hdd  0.09769          osd.17         up   1.00000  1.00000
18    hdd  0.09769          osd.18         up   1.00000  1.00000
19    hdd  0.09769          osd.19         up   1.00000  1.00000
-7         0.97687      host ubuntu03
20    hdd  0.09769          osd.20         up   1.00000  1.00000
21    hdd  0.09769          osd.21         up   1.00000  1.00000
22    hdd  0.09769          osd.22         up   1.00000  1.00000
23    hdd  0.09769          osd.23         up   1.00000  1.00000
24    hdd  0.09769          osd.24         up   1.00000  1.00000
25    hdd  0.09769          osd.25         up   1.00000  1.00000
26    hdd  0.09769          osd.26         up   1.00000  1.00000
27    hdd  0.09769          osd.27         up   1.00000  1.00000
28    hdd  0.09769          osd.28         up   1.00000  1.00000
29    hdd  0.09769          osd.29         up   1.00000  1.00000
root@ubuntu01:/#
---


## Pool Creation

### Replicated Pool
```bash
ceph osd pool create case_study 64 64 replicated
ceph osd pool set case_study size 3
ceph osd pool application enable case_study rbd
```

### Erasure Coded Pool
```bash
ceph osd erasure-code-profile set fatih_policy_ec k=8 m=2 crush-failure-domain=host
ceph osd pool create my_ec_pool 64 64 erasure fatih_policy_ec
ceph osd pool application enable my_ec_pool rgw
```

> 🧠 `k=8 m=2` means 8 data chunks + 2 coding chunks. Minimum 8 OSDs needed to write, 9 to recover.

---

## Cluster Status Checks

### Cluster Status
```bash
ceph -s
```

### Pool Detail
```bash
ceph osd pool ls detail

pool 1 '.mgr' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 1 pgp_num 1 autoscale_mode on last_change 20 flags hashpspool stripe_width 0 pg_num_max 32 pg_num_min 1 application mgr read_balance_score 30.00
pool 2 'case_study' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 64 pgp_num 64 autoscale_mode on last_change 218 flags hashpspool stripe_width 0 read_balance_score 2.34
pool 3 'my_ec_pool' erasure profile fatih_policy_ec size 10 min_size 9 crush_rule 1 object_hash rjenkins pg_num 64 pgp_num 64 autoscale_mode on last_change 221 flags hashpspool,creating stripe_width 32768
```


## Version Check

```bash
ceph version
ceph version 18.2.4 (e7ad5345525c7aa95470c26863873b581076945d) reef (stable)
ceph versions
{
    "mon": {
        "ceph version 18.2.4 (e7ad5345525c7aa95470c26863873b581076945d) reef (stable)": 3
    },
    "mgr": {
        "ceph version 18.2.4 (e7ad5345525c7aa95470c26863873b581076945d) reef (stable)": 2
    },
    "osd": {
        "ceph version 18.2.4 (e7ad5345525c7aa95470c26863873b581076945d) reef (stable)": 30
    },
    "overall": {
        "ceph version 18.2.4 (e7ad5345525c7aa95470c26863873b581076945d) reef (stable)": 35
    }
}
```

Expected Output:
```json
"mon": {
  "ceph version 18.2.4 reef": 3
},
"mgr": {
  "ceph version 18.2.4 reef": 2
},
"osd": {
  "ceph version 18.2.4 reef": 30
}
```

---

## Upgrade mon and mgr to Squid (v19.x)

### Upgrade Command
```bash
ceph orch upgrade start --image quay.io/ceph/ceph:v19.2.1 --services mon,mgr
https://docs.ceph.com/en/latest/releases/#active-releases
```

### Post-Upgrade Checks
```bash
ceph versions
```
Expected Output:
```json
"mon": {
  "ceph version 19.2.1 squid": 3
},
"mgr": {
  "ceph version 19.2.1 squid": 2
},
"osd": {
  "ceph version 18.2.4 reef": 30
}
```

You can verify daemon versions:
```bash
ceph orch ps --daemon-type mon
NAME          HOST      PORTS  STATUS        REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID
mon.ubuntu01  ubuntu01         running (4m)     4m ago   2d    26.5M    2048M  19.2.1   f2efb0401a30  7e8556038765
mon.ubuntu02  ubuntu02         running (4m)     4m ago   2d    17.6M    2048M  19.2.1   f2efb0401a30  2a80480069ff
mon.ubuntu03  ubuntu03         running (3m)    50s ago   2d    33.3M    2048M  19.2.1   f2efb0401a30  26116309b7c3
ceph orch ps --daemon-type mgr
NAME                 HOST      PORTS             STATUS        REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID
mgr.ubuntu01.iwjidf  ubuntu01  *:8443,9283,8765  running (4m)    85s ago   2d     543M        -  19.2.1   f2efb0401a30  ad9d2b05a71c
mgr.ubuntu02.wuifuw  ubuntu02  *:8443,9283,8765  running (3m)     2m ago   2d     477M        -  19.2.1   f2efb0401a30  a28a6757eb20

```

---

## References
- [Ceph PG Calculator](https://ceph.io/pgcalc/)
- [Ceph Reef to Squid Upgrade Docs](https://docs.ceph.com/en/reef/cephadm/upgrade/)
- [Ceph Pool Types](https://docs.ceph.com/en/latest/rados/operations/pgcalc/)

