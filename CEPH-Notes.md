# Cheat Sheet

## Shorts:
- OSD - 1 deamon per 1 disk 
- CRUSH - algorithm to store and retrieve data + monitor and so
- RADOS - old name for CEPH
- Pools - dedicated containers to store objects based on usage
- PG - placement groups - Help to map OSDs into the containers and used in replication
- PGP - placement groups with pools - number of placement groups based on pools
- Object - Chunk of data + metadata in one place
- Journal - dedicated mostly SSD disk to handle spikes and after we copy over the objects to OSD disk
- Replica - number of copies for object to ensure not lossing data based on replica factor count

### Pools 
- Logical splitting of storage (like dirs or tenants)
- We create per user, or per data, or per usage
- Pools have defined type defines data durability and it is for full pool, it is defined during creation
- Pools has a lot of PGs which are storing data

### Placement Groups (PG)
- CRUSH alogirthm works per PG, not per object -> this is more efficient
- It holds a set of OSDs, where data will be saved
- Number of PGs is defined during pool creation
- When we need to move a lot of objects,  CRUSH use per PG
- Too few number of PGs can cause moving a lot of objects, not effective
- Too less number of PGs, can be very CPU intensive, not effective
- Expanding the cluster - new OSDs wil get few PGs during rebalancing
- Shrinking the cluster - loosing some OSD will cause that misisng PGs will be recalculated and created on other PGs

### CRUSH rulesets
- Assignet to pool
- Identifies failure domains and performance domains for the pool also finds primary OSD for PG for Read/Write
- Allows to add OSDs into the hierarchy like Racks, DCs, Server, ...
- Copy objects acros failure domains
- Allows to define multiple types of disks SSD, HD, ... based on performance 
- Create replicas of the object
- It is running on OSDs

## Client side notes
- Clients contacts cephmons to get the latest cluster-map (CRUSH ruleset)
- After it knows where the primary OSD is located for specific pools
- In this case it can do I/O to the pools
- Client needs to know only object ID and pool name
- Crush during write first calculate a hash  from Object ID and pool name to get PG
- librados is the main library, which suggested to use as allow to compute Hash for PG on client side

1. The client inputs the pool ID and the object ID. For example, pool = liverpool and object-id =
john.
2. CRUSH takes the object ID and hashes it.
3. CRUSH calculates the hash modulo of the number of PGs to get a PG ID. For example, 58.
4. CRUSH calculates the primary OSD corresponding to the PG ID.
5. The client gets the pool ID given the pool name. For example, the pool "liverpool" is pool number
6. The client prepends the pool ID to the PG ID. For example, 4.58.
7. The client performs an object operation such as write, read, or delete by communicating directly
with the Primary OSD in the Acting Set.


- When client writes to the main OSD, that OSD will replicate to other OSDs based on replication count
- After we confirm that write was done
- Replication is done by CRUSH which creates rules to do copies over failure domains 

- All clients use the RADOS protocol to communicate with CEPH
- Client requirements:
  - Ceph config file and Ceph monitor addresses
  - Pool Name
  - Username and path to secret key

- Client can register a watch to persistently monitor an object and keep session to primary OSD
- Client can send a notify, which will make OSD to inform every client who is watching the same object

- `Mandatory Exclusive Locks` allows to client to lock an RBD and it helps against write hazards when more client mount the same RBD
- Other clients will check before write if disk is locked or not
- Not enabled by default
- enable by --image-feautre 5 (1+4, 1=Enables layering, 4=Enables locks)

- `Object Mapping` if enabled, allows client to track already written objects so less query is required to cluster
- Object Map is tracked in memory
- It can be used in resize, export, flatten, delete, read, ...
- Enabled by default
- If we want to enable with excluse locks we need:
- enable by --image-feautre 13 (1+4+8, 1=Enables layering, 4=Enables locks, 8=Enables Object Mapping)

- `Object Stripping` - split data into multiple objects and save it separately
- Better performance for large files
- We distribute the objects over multiple PGs (OSDs) which does not limit the speed of one disk
- Object Size = how much data we can have per object (4MB - 16MB), 16 is recomended
- Strip Size = how much strips we have per object, example 64KB, so it will have enough strips per Object
- Strip Count = Write sequence over the series of objects after it returns to the first object (like we are stripping to 4 objects)

### Client Storage Options
- RBD - Rados Block Device = Block storage option like a remote raw disk (like iSCSI disks)
- CephFS - Ceph FileSystem = Storage as file system which can be remotely mounted (like NFS)
- Object Storage = Save data via Restful API to Ceph (like S3 in AWS)

## CEPH Manager
- Handles read operation and it has all info about the cluster
- Greatly reduce the load required for data collection for OSDs

## Counting the PGs:

Total PGs = ((Total_number_of_OSD * 100) / max_replication_count) / pool count

## Ceph structure:

Pools — holds —> PG — has couple. —> OSDs

We can have dedicated pools just with SSD disks or with mixed disks

## Ceph Data management:
First data is written into primary OSD then its replicated to secondaries.
Secondaries sends ack, and when primary receive all ack, it sends ack to client

## Eraure Coding
- Ensure that lost data can still be recovered
- N = K + M
  - N = total word (object), data chunks combined
  - K = data chunks (original data)
  - M = coding objects to ensure protection agains data loss
- 10/16 = K=10, M=6, N16
  - This will spread 16 chunks over OSDs
  - M = 6, allows to loose 6 OSDs without data loss
- Administrators have 5 OSDs and 2 can be lost -> M=2

## Object Store
- Low level interface to OSDs block device
- Ensure:
  - Atomicity = Transactions are all or nothing saved and after confirmed
  - Consistency = ensure CEPH semantics are correct
  - Isolation = Invoke Sequencer to ensure that writes are in correct order
  - Durability = ensured by Erasure Coding for ACID transactions
- Ceph implements following methods to store data: (* most common)
  - Filestore* = use filesystem
  - BlueStore* = production grade to utilize raw block devices
  - MemStore = in memory save
  - K/V Store = internal Key/Value database imeplenetation

### FileStore
- Imeplements mostly as XFS
- Object is saved to disk via filesystem
- PGs = Directories, Objects = Files, Metadata = XATTr
- In past btfrs was considered as next step but it never reached the reliability requirements

### BlueStore
- Next generation, production grade implementation
- Improvement to facility the SSD and NVMe impelmenation
- Uses very ligthweight BlueFS
- Eliminates double write penalty
- Objects are stored directly as blocks on raw disk - no FileSystem
- Block Database = Key/Value DB, ObjectID=Key, Value=Series of block addresses for stored data
- DB save checksum for every data/metadata and it is checked during every read
- Supports Data compression 
- WriteAhead - ensure that data is written directly to disk with Atomicity via logs (WAL), no double write to journal

## CEPH rebalancing and recovery
- If new OSD is added or deleted
- Cluster map is updated by CEPH
- CRUSH recalculates the cluster and place data evenly
- If we have 50 OSDs and we add 1 = we will move 1/50 = 2% of data
- Many of the PGs remains in the same place and we move only necessery so no spikes in resources

## Data Integrity
- `Scrubbing `
  - Scrubbing objects in PG, Compares object Metadata with one PG, with others in secondary PGs
  - Performed daily
  - Catches bugs or storage errors
  - Deep Scrubbing - compares data objects bit by bit, it catches bad disk sectors, performed usually weekly

- `CRC Checks`
  - Only in bluestore
  - During write we save CRC of object to DB
  - During read we always compare if its matching with block DB and our calculated

## Checking Commands:
```# ceph osd status```
- Check if CEPH cluster is healthy

```# ceph osd tree```
- Check osd status, if they are up and in which tree located

```# rados lspools```
- List of pools

```# ceph  osd lspools```
-  List pools (same as before)

```# ceph osd pool ls```
- List pools (same as before)

```# ceph osd pool get data pg_num```
```# ceph osd pool get <poolname> pgp_num```
- Get pg and pgp numbers 

```# ceph osd dump```
- Get replication factor per pools

```# ceph osd pool ls detail```
- Something like dump but show only pools

## Create Pool

```# ceph osd pool create web-services 128 128```
- Create pool with name web-services with 128 PG  and 128 PGP

```# ceph osd pool set web-services size 3```
- Set replica factor to 3

```# rados -p web-services put object 1 /tmp/index.html```
- Save index.html file to ceph as object 1

```# rados -p web-services ls```
- List objects in pool

```# ceph osd map web-services object1```
 - Check the object directly inside the pool

```# ceph osd pool renanme web-serves frontend-web```
- Rename pool

```# ceph osd pool mksnap frontend-web snapshot-from-fronend ```
- Make a  snapshot

```# rados lssnap -p frontend-web```
- List snapshots

## HW Requirements

### Cephmon
- Entry level server is enough
- 1 Core, few Gig memory server
- Non production can use VMS
- In production typically its a dedicated low cost server (rasperry pi 4 can be used? )

### OSDs
- 1 OSD per physical disk (or 1 per raid but RAID is not suggested)
- 1Ghz of CPU and 2GB of RAM per OSD
- Higher disks may need more RAM
- Separate Journal disk for OSD (multiple OSDs can use the same journal)

### MDS
- More resources then Cephmon
- 4 cores at least a lot of memory, more memory better performance
- Dedicated Physical Server

### Other
- Required to have a dedicated storage network with at least 1Gbps but 10G is advised
- Suggested to have network redundancy

## Install
- Check ceph-deploy

## Upgrade
1. Upgrade Ceph Monitors
2. Upgrade OSDs


# Deployment options
------------------------
## Requirements
- 2 Core CPU
- 8GB RAM
- Virtualization enabled in BIOS

## CEPH-Deploy
- main tool to deploy CEPH is via CEPH Deploy
- require passwordless SSH to nodes

## Ansible
- Supported and possible to deploy via Ansible
- Pre-prepared roles and playbooks
- Faster and reusable deployments
- Kind of orchestrated deployment
- Check documentation page

Example of deployment:
/etc/ansible/group_vars/ceph:
```
ceph_origin: 'repository'
ceph_repository: 'community'
ceph_mirror: http://download.ceph.com
ceph_stable: true # use ceph stable branch
ceph_stable_key: https://download.ceph.com/keys/release.asc
ceph_stable_release: mimic # ceph stable release
ceph_stable_repo: "{{ ceph_mirror }}/debian-{{ ceph_stable_release }}"
monitor_interface: enp0s8 #Check ifconfig
public_network: 192.168.0.0/24
```

/etc/ansible/group_vars/osds:
```
osd_scenario: lvm
lvm_volumes:
- data: /dev/sdb
```

Run:
```# ansible-playbook -K site.yml```


## Containers
- Most popular is Rook project
- We can deploy CEPH services as containers and the best is if Kubernetes is managing it

Install:
1. Install Docker
1. Start/Enable Docker
1. Install Kubernetes
1. Init Kubernetes cluster
1. Git clone Rook
1. Create Containers via kubectl create in K8s cluster
