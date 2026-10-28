# 3. Base OS setup

Everything here runs on the instance (`ssh <host>`).

## Updates

```
sudo apt-get update
sudo apt-get -y full-upgrade
sudo reboot
```

Reconnect after about 30 seconds. Ubuntu installs security updates on its own through
`unattended-upgrades`, which is on by default. Check that it is:

```
systemctl status unattended-upgrades --no-pager
```

When a login message says `*** System restart required ***`, run `sudo reboot` at a quiet moment.

## Hostname and timezone

```
sudo hostnamectl set-hostname <host>
sudo timedatectl set-timezone <timezone>
```

Keeping UTC is fine too; logs from containers are in UTC either way.

## Swap

Swap keeps a memory spike (a big build, a model load) from getting a process
OOM-killed. It's slower than RAM, so `docker stats` should still be the guide for sizing.

```
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
free -h
```

`swappiness=10` makes the kernel prefer RAM and swap only under pressure.

## Mount the data volume

Find the data volume. It's the 100G disk with no partitions and no filesystem (empty `FSTYPE`):

```
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
```

Usually it's `nvme1n1`, but **check the size**: `mkfs` erases whatever is on the disk. Skip the
`mkfs` line if the volume already has a filesystem (for example, one moved over from another
instance).

```
sudo mkfs.ext4 -L data /dev/nvme1n1
sudo mkdir -p /data
echo 'LABEL=data /data ext4 defaults,noatime,nofail 0 2' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
sudo chown ubuntu:ubuntu /data
df -h /data
```

- Mounting by `LABEL` survives device renames (NVMe names aren't guaranteed to be stable).
- `nofail` lets the instance still boot if the volume is detached.
- `ubuntu` is uid 1000, which matches the non-root user in many Docker images, so
  bind mounts under `/data` are writable from containers.

### Growing it later

After **EC2 → Volumes → Modify** to a bigger size:

```
sudo resize2fs /dev/nvme1n1
```

No reboot or unmount needed.
