# ZFS Server Setup Guide - Optimal Configuration

## Overview & Design Decisions

### Recommended Architecture

- **Primary Storage**: ZFS RAIDZ1 pool using sdb, sdc, sdd (3x 10TB drives)
- **Cache & Performance**: NVMe drive partitioned for ZFS cache and databases
- **Backup Target**: Existing 8TB drive (sda) for nightly ZFS snapshots
- **Boot Drive**: Keep existing Ubuntu installation on sdf untouched

### Why This Setup?

1. **ZFS as Primary**: Better data integrity, snapshots, compression, and built-in RAID
2. **RAIDZ1**: Can lose 1 drive without data loss, good balance of capacity vs redundancy
3. **NVMe Caching**: L2ARC cache dramatically improves read performance
4. **8TB as Backup**: Reliable backup destination using ZFS send/receive

## Step-by-Step Implementation

### Phase 1: Prepare the System

#### 1.1 Install ZFS

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install ZFS
sudo apt install zfsutils-linux -y

# Verify installation
sudo zpool status
```

#### 1.2 Backup Existing Data

```bash
# Create temporary backup of docker configs
sudo cp -r ~/apps/docker/compose ~/apps/docker/compose_backup_$(date +%Y%m%d)

# Verify current 8TB drive health
sudo smartctl -a /dev/disk/by-uuid/1d590c2f-6937-4136-b690-c91895ad999f
```

### Phase 2: Identify Drive UUIDs

#### 2.1 Get Drive Serial Numbers and Create Persistent Names

```bash
# Get drive information
sudo lsblk -o NAME,SIZE,SERIAL,MODEL

# For each 10TB drive, get the serial number and create by-id links
ls -la /dev/disk/by-id/ | grep -E "(sdb|sdc|sdd)"

### Output:
# > ls -la /dev/disk/by-id/ | grep -E "(sdb|sdc|sdd)"
# lrwxrwxrwx 1 root root    9 Jun 13 12:47 ata-HUH721010ALE601_1SJ2X5DZ -> ../../sdc
# lrwxrwxrwx 1 root root    9 Jun 13 12:47 ata-HUH721010ALE601_2TG87Y5D -> ../../sdb
# lrwxrwxrwx 1 root root    9 Jun 13 12:47 ata-HUH721010ALE601_2TJML5ZD -> ../../sdd

# Note the full by-id paths - they look like:
# /dev/disk/by-id/ata-WDC_WD101EFAX-68LDBN0_WD-WX12345678
```

### Phase 3: Partition the NVMe Drive

#### 3.1 Backup and Repartition NVMe

```bash
# Backup any important data from nvme partitions first!
# Then unmount existing partitions
sudo umount /mnt/nvme

# Remove from fstab temporarily
sudo sed -i '/762fdaf4-85a8-42b1-a781-0704878f4ff5/d' /etc/fstab

# Partition the NVMe drive
sudo parted /dev/nvme0n1 --script -- \
  mklabel gpt \
  mkpart primary 1MiB 50GiB \
  mkpart primary 50GiB 100GiB \
  mkpart primary 100GiB 100%

# Wipe filesystem signatures from the new partitions
sudo wipefs -a /dev/nvme0n1p1
sudo wipefs -a /dev/nvme0n1p2
sudo wipefs -a /dev/nvme0n1p3

# Set partition types
sudo parted /dev/nvme0n1 set 1 raid on  # For ZFS L2ARC cache
sudo parted /dev/nvme0n1 set 2 raid on  # For ZFS ZIL (log)
# Partition 3 will be for databases

# Format database partition
sudo mkfs.ext4 -L "storage-nvme" /dev/nvme0n1p3

# Get new UUIDs
sudo blkid /dev/nvme0n1p3

# Output:
# /dev/nvme0n1p3: LABEL="storage-nvme" UUID="9fe7a886-f0c2-4940-8fa3-02d871636641" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="primary" PARTUUID="5c003db1-7954-4ab1-b88d-db21fde398a1"
```

### Phase 4: Create ZFS Pool

#### 4.1 Create the Main Storage Pool

```bash
# Replace with your actual by-id paths from step 2.1
ZFS_DRIVES=(
  "/dev/disk/by-id/ata-HUH721010ALE601_2TG87Y5D"
  "/dev/disk/by-id/ata-HUH721010ALE601_1SJ2X5DZ"
  "/dev/disk/by-id/ata-HUH721010ALE601_2TJML5ZD"
)

# Create RAIDZ1 pool
sudo zpool create -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O xattr=sa \
  -O recordsize=1M \
  storage raidz1 "${ZFS_DRIVES[@]}"

# Add cache and log devices
sudo zpool add storage cache /dev/nvme0n1p1
sudo zpool add storage log /dev/nvme0n1p2

# Verify pool creation
sudo zpool status storage
sudo zfs list
```

#### 4.2 Create ZFS Datasets

```bash
# Create top-level datasets
sudo zfs create -o recordsize=128K storage/docker
sudo zfs create -o recordsize=128K storage/backups

# Create unified datasets for each application (config + media together)
sudo zfs create -o recordsize=1M storage/docker/immich      # Photos benefit from large recordsize
sudo zfs create -o recordsize=1M storage/docker/nextcloud   # Mixed content, 1M is good default
sudo zfs create -o recordsize=1M storage/docker/plex        # Large media files
sudo zfs create -o recordsize=1M storage/docker/servarr     # Mixed: small configs + large media
sudo zfs create -o recordsize=128K storage/docker/heimdall  # Mostly small files
sudo zfs create -o recordsize=128K storage/docker/netdata   # Monitoring data

# Set mountpoints
sudo zfs set mountpoint=/storage storage
sudo zfs set mountpoint=/storage/docker storage/docker
sudo zfs set mountpoint=/storage/backups storage/backups
sudo zfs set mountpoint=/storage/docker/immich storage/docker/immich
sudo zfs set mountpoint=/storage/docker/nextcloud storage/docker/nextcloud
sudo zfs set mountpoint=/storage/docker/plex storage/docker/plex
sudo zfs set mountpoint=/storage/docker/servarr storage/docker/servarr
sudo zfs set mountpoint=/storage/docker/heimdall storage/docker/heimdall
sudo zfs set mountpoint=/storage/docker/netdata storage/docker/netdata
```

#### 4.3 Delete ZFS datasets (additional information)

For example, if you need to delete a dataset /storage/test:

```bash
sudo zfs destroy storage/test # To delete the dataset
sudo rm -rf /storage/test # If you need to remove the mount point as well
```

### Phase 5: Mount Database Partition

#### 5.1 Setup Database Storage

```bash
# Get UUID of database partition
DB_UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p3)

# Create mount point and mount
sudo mkdir -p /storage/nvme
sudo mount UUID=$DB_UUID /storage/nvme

# Add to fstab
echo "UUID=$DB_UUID /storage/nvme ext4 defaults,noatime 0 2" | sudo tee -a /etc/fstab

# Create directories for different databases
sudo mkdir -p /storage/nvme/{postgres,mysql,redis}
sudo chown -R 1000:1000 /storage/nvme
```

### Phase 6: Migrate Docker Applications

#### 6.1 Stop Services and Migrate Data

```bash
# Navigate to docker compose directory
cd ~/apps/docker/compose

# Stop all services
for dir in */; do
  if [ -f "$dir/docker-compose.yml" ] || [ -f "$dir/compose.yml" ]; then
    echo "Stopping $dir"
    cd "$dir" && docker-compose down && cd ..
  fi
done

# Create new directory structure
sudo mkdir -p /storage/docker/{immich,nextcloud,plex,servarr,heimdall,netdata}
sudo chown -R 1000:1000 /storage/docker/

# Copy existing docker configs to new locations
sudo cp -r ~/apps/docker/compose/* /storage/docker/
```

#### 6.2 Update Docker Compose Files

```bash
# You'll need to update each docker-compose.yml file to use new paths
# Example for immich (repeat for each service):

# Create updated compose file template
cat > /storage/docker/immich/docker-compose.yml << 'EOF'
version: '3.8'

services:
  immich-server:
    # ... existing configuration ...
    volumes:
      - /storage/docker/immich/config:/config
      - /storage/media/photos:/photos
      - /storage/nvme/postgres/immich:/var/lib/postgresql/data
    # ... rest of configuration ...
EOF

# Repeat this process for each service, updating volume paths appropriately
```

### Phase 7: Setup Automated Backups

#### 7.1 Create Backup Scripts

```bash
# Create backup script directory
sudo mkdir -p /opt/backup-scripts

# Create ZFS backup script
sudo tee /opt/backup-scripts/zfs-backup.sh << 'EOF'
#!/bin/bash

POOL="storage"
BACKUP_LOCATION="/mnt/storage/zfs-backups"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/zfs-backup.log"

# Ensure backup location exists
mkdir -p "$BACKUP_LOCATION"

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

log_message "Starting ZFS backup process"

# Create snapshots
for dataset in $(zfs list -H -o name | grep "^$POOL/"); do
    snapshot_name="${dataset}@backup-${DATE}"
    log_message "Creating snapshot: $snapshot_name"
    zfs snapshot "$snapshot_name"
done

# Send incremental backups
for dataset in $(zfs list -H -o name | grep "^$POOL/" | grep -v "@"); do
    dataset_name=$(basename "$dataset")
    backup_file="$BACKUP_LOCATION/${dataset_name}-${DATE}.zfs"

    # Find the most recent backup
    latest_snapshot=$(zfs list -t snapshot -H -o name | grep "$dataset@backup-" | tail -n2 | head -n1)
    current_snapshot=$(zfs list -t snapshot -H -o name | grep "$dataset@backup-" | tail -n1)

    if [ -n "$latest_snapshot" ] && [ "$latest_snapshot" != "$current_snapshot" ]; then
        # Incremental backup
        log_message "Creating incremental backup: $dataset ($latest_snapshot to $current_snapshot)"
        zfs send -i "$latest_snapshot" "$current_snapshot" | gzip > "$backup_file.gz"
    else
        # Full backup
        log_message "Creating full backup: $dataset ($current_snapshot)"
        zfs send "$current_snapshot" | gzip > "$backup_file.gz"
    fi
done

# Clean up old snapshots (keep last 7 days)
for snapshot in $(zfs list -t snapshot -H -o name | grep "@backup-" | head -n -7); do
    log_message "Removing old snapshot: $snapshot"
    zfs destroy "$snapshot"
done

# Clean up old backup files (keep last 30 days)
find "$BACKUP_LOCATION" -name "*.gz" -mtime +30 -delete

log_message "ZFS backup process completed"
EOF

# Make script executable
sudo chmod +x /opt/backup-scripts/zfs-backup.sh
```

#### 7.2 Setup Cron Job

```bash
# Add nightly backup job
echo "0 2 * * * root /opt/backup-scripts/zfs-backup.sh" | sudo tee -a /etc/crontab

# Test the backup script
sudo /opt/backup-scripts/zfs-backup.sh
```

### Phase 8: Performance Tuning

#### 8.1 ZFS Performance Optimization

```bash
# Set ARC (cache) limits - adjust based on your 32GB RAM
echo 'options zfs zfs_arc_max=17179869184' | sudo tee -a /etc/modprobe.d/zfs.conf  # 16GB max
echo 'options zfs zfs_arc_min=8589934592' | sudo tee -a /etc/modprobe.d/zfs.conf   # 8GB min

# Enable some performance features
sudo zfs set primarycache=all storage
sudo zfs set secondarycache=all storage
sudo zfs set logbias=throughput storage/media
sudo zfs set logbias=latency storage/databases
```

#### 8.2 System Optimization

```bash
# Add optimizations to sysctl
sudo tee -a /etc/sysctl.conf << 'EOF'
# ZFS optimizations
vm.swappiness=1
vm.vfs_cache_pressure=50
vm.dirty_ratio=5
vm.dirty_background_ratio=2
EOF

# Apply changes
sudo sysctl -p
```

### Phase 9: Start Services

#### 9.1 Start Docker Services

```bash
# Navigate back to docker directory
cd /storage/docker

# Start services one by one to verify they work
for dir in */; do
  if [ -f "$dir/docker-compose.yml" ] || [ -f "$dir/compose.yml" ]; then
    echo "Starting $dir"
    cd "$dir" && docker-compose up -d && cd ..
    sleep 10  # Give time for service to start
  fi
done
```

### Phase 10: Monitoring and Maintenance

#### 10.1 Create Health Check Script

```bash
sudo tee /opt/backup-scripts/zfs-health-check.sh << 'EOF'
#!/bin/bash

LOG_FILE="/var/log/zfs-health.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Check pool status
POOL_STATUS=$(zpool status storage | grep "state:" | awk '{print $2}')

if [ "$POOL_STATUS" != "ONLINE" ]; then
    log_message "WARNING: ZFS pool status is $POOL_STATUS"
    # You could add email notification here
fi

# Check disk usage
zfs list -o name,used,available,mountpoint

# Check for scrub status
zpool status storage

log_message "Health check completed"
EOF

sudo chmod +x /opt/backup-scripts/zfs-health-check.sh

# Add weekly health check
echo "0 8 * * 0 root /opt/backup-scripts/zfs-health-check.sh" | sudo tee -a /etc/crontab
```

#### 10.2 Setup Automatic Scrub

```bash
# Add monthly scrub job
echo "0 1 1 * * root zpool scrub storage" | sudo tee -a /etc/crontab
```

## Usage and Maintenance

### Daily Operations

- Monitor disk usage: `zfs list`
- Check pool status: `zpool status`
- View recent snapshots: `zfs list -t snapshot`

### Backup Restoration

```bash
# To restore from backup
gunzip -c /mnt/storage/zfs-backups/dataset-name-DATE.zfs.gz | zfs receive storage/restored-dataset
```

### Adding New Services

1. Create new ZFS dataset: `sudo zfs create storage/docker/new-service`
2. Set appropriate recordsize based on service type
3. Create docker-compose.yml with proper volume mappings

### Performance Monitoring

- Check ARC usage: `cat /proc/spl/kstat/zfs/arcstats`
- Monitor I/O: `zpool iostat storage 1`
- Check fragmentation: `zpool list -o fragmentation storage`

## Troubleshooting

### Common Issues

1. **Pool won't import**: Use `zpool import -f storage`
2. **Poor performance**: Check ARC settings and consider adding more cache
3. **Snapshot space issues**: Clean up old snapshots regularly
4. **Database performance**: Ensure databases are on NVMe partition

### Recovery Procedures

- Boot from USB if needed to import pool with `zpool import -R /mnt storage`
- Use ZFS send/receive for data migration between pools
- Always test backup restoration procedures regularly

---

**Note**: This setup provides excellent performance, redundancy, and backup capabilities. The NVMe caching will significantly improve read performance, while the RAIDZ1 setup protects against single drive failure. Regular snapshots and backups to the 8TB drive ensure data safety.
