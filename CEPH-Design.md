# CEPH Design Notes

## General
-----------------------------

### SSD
- Read and Write are in micro seconds
- Overwrite a block (128K) is slower,  it needs  Read, Erase and Write - several miliseconds
- They need RAM Buffer = volatile, to avoid overwrite limitation
- Power Loss can be achieved by high capacitors on SSD, so if they loose power, it can still flush data from RAM buffer
- We should choose enterprise level SSDs so we have expected performance
- Low level SSD can make untested bugs, low speed and data loss
- Enterprise SSD guarantee that data was written when it replies
- Enterprise Read Intensive SSDs have ususally between 0.3 - 1 data writes per day 
- So 480GB data written daily to 480GB SSD will make SSD 5 year lifetime, 960GB daily half of it, 2.5 year
- Enterprise General usage have higher 3-5 Data per day write (DPDW)
- Enterprise Write intensive are the most expensive 10+ DPDW, recommeneded for the Journals
- Journal will handle writes before it is written to disk
- It should handle 2x number of write for expected performance

### Disk
- Go for IOPs and not capacity
- Go for Hot Swap disks, so node will remain UP during disk replacement

### Memory
- Bluestore ODS recommendation but its variable, this is bare minimum
  - 4 GB for every HDD OSD
  - 5 GB for every SSD OSD
- More memory is just better performance
- Main memory affect is number of PGs running on OSD
- If cluster is running less then 200PGs per OSD, less then 4GB RAM is enough per OSD
- If PG number is higher we need more memory
- If OSD is removed, PGs are redistributed to existing OSDs
- Larrge swap on SSD is recomended to reduce OOM kill OSD in case of low memory issues
- ECC Memory should be used always

### CPU
- Official recomendation is 1GHZ for OSD (good for HDD)
- But its I/O related and I/O size, which needs more and more CPU
- If not enough CPU resources, it can start timing out and getting marked out from Cluster
- Around 10MHZ is recommended per I/O (needs testing)
- If we run out of CPU, the ODS will disconnect intermittently from Cluster
- High performance deployment needs at least 3GHZ multicore CPU and we need NUMA consideration for multi socket motherboards

### Networking
- Go for 10G links, 1G has poor performance for OSDs, not sufficient for production grade
- 1G traffic can also cause issues with I/O and even OSD can loose connectivity
- We have north-west for client to ODS traffic and east-west for ODS to OSD traffic
- If multiple ports available, go for high availability = Port Channels to use all avaialable bandwith
- Leaf/Spine is preffered

### Failure scenarios
- CEPH is designed to survive loosing OSD, OSDs or even full servers
- Lossing 25%  of  disks is still ok, we  can recover all data just it will take few days if we have several TB of disks
- CEPH preffers to not loose any data over availability

### File Systems
- Data and Metadata must be carried over single transaction for 1 object!
- BTFRS was promising to do ACK after couple of transactions not 1-1, but it failed due to other limits
- XFS became the standard but that does not support transactions
- Because of missing support for transactions, we use ahead journaling
- There is penalty if we have more journals in 1 disk
- Power Loss protection require more advanced/expensive disks with capacitors
- XFS also limits file size (object size) and number of files (Objects) per disk
- EOL

- `BlueStore` = fixing the XFS limitation
- Using RocksDB to save metadata + data blobs info
- Block devices to save data directly on disk
- Each object is allocated as X amount of blobs on block device
- RocksDB is a high performance key-value store
- BlueStore is using compression at blob level (X blobs = 1  object) = snappy, zlib, zstd
- Compression ratio is by default 87,5 = can be tunned
- Checksum - calculated during write, every data during read is compared, if checksum is not matching, data is fixed 
- Supports auto-tune of memory, starting by 4G/OSD and increasing if possible
- The extended memory is cache, we can preallocate the cache and it will increase/decrease for OSDs based on need
- Most of data writes are written to disk directly, no need for double writes
- `BlueFS` = minimal FS to write data/metadata to disk
- You cannot read by mounting, but you can use mount to fix errors
- `ceph-volume` = tool to provision blueStore OSD
- Example to provision disk:
```ceph-volume create --bluestore /dev/sda --block.wal /dev/sdb --block.db /dev/sdc (--dmcrypt)```
   - --dmcrypt add support for disk encryption = recomended
   - /dev/sda = slower disk 
   - /dev/sdb = faster diks (SSD)
   - /dev/sdc = fast disk (NFVMe)

#### Upgrade to BlueStore

- Upgrading to BlueStore is straightforward, destroy the OSD and create new as BlueStore
- Degraded - just destroy and wait for automatic fix
- Out and In - tell other OSDs they you are going off, they will rebalance  PGs and  you can replace

Steps:
- Stops OSDs
- umount the XFS
- format the disk: `sudo ceph-volume lvm zap <disk>`
- check disks and type to linux system by fdisk
- remove old OSDs, where X is the name from ceph osd tree: `sudo ceph osd purge X --yes-i-really-mean-it`
- Create encrypted OSDs as BlueStore: `sudo ceph-volume lvm create --bluestore --data /dev/sd<x> --block.db /dev/sda<ssd> --dmcrypt`
- Start and Enable service: systemctl start/enable ceph-osd@X 
- Wait for health_ok and continue witth next node

### Other
- Price of server and disks needs to be considered
- Power loss protection via more power sources or power supplies

## Deployment
-----------------------

### Before Deployment
- Identify the use case and choose the appropriate HW
- Most common use cases:
  - Throughput optimized - Fast data access, SSDs are required, typicially streaming (cameras)
  - Capacity optimized - Focus on huge space (terabytes), SSD for journaling, storing data, archiving, backups
  - IOPS optimized - high R/W workload, SSD and NVMe storage is required
  
- Data Durability is important factor, Erasure Coding helps to lower the copies per object and not making redundant copies
- For Erasure coding we need as many nodes as K+M at least.
- If it is multisite deployment, place data close to clients if possible
- Power supplies and Power Sources idealyy should be redundant

### Tunning the Kernel
- vm.min_free_kbytes - sysctl.conf - hard allocation of memory to OSDs (for every 64GB, reserver 1GB)
- /etc/security/limits.conf - increase the file descriptors to not run out of it -> unlimited

### Recomendations
- Dont use low grade SSDs
- If RAID is used, use battery for data protection
- Don't configure unknown options
- Use 10G NICs
- Have a backup and recovery plan
- Test failure scenarios (lossing disks, node, ...)
- Choose better hardware (more tested) Idealy supported by RedHat
- Learn, keep on track, subscribe to emails/IRC/...
- Build POC to see if it will be usable in bigger deployments
- Replication (size) should  be at least 3+ and min_size 2+
- min_size controls how many replica needs to be written so we ACK to client
- avoid min_size 1

### Define goals
Examples:
- It should cost less then defined amount
- It should meet the target performance (IOPS & MB/sec)
- It should survive failure scenarios


