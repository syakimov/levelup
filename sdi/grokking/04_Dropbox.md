*Assumptions*
500M total userss
100M daily active (DAU)
each user connects from three devices
a user has 200 files
each file is 100KB
1Million active connections per second

1. Problems that Cloud Storage Solves
  - Availability - anywhere anytime
  - Reliability and Durability
  - Scalability

2. Requirements and Goals of the System
  - upload and download their files/photos from any device.
  - share files or folders with other users.
  - automatic synchronization between devices
  - user storage limit 1GB
  - ACID-ity is required
  - offline editing

  - Extended Requirements
      support snapshots and versioning

3. Design Considerations
    - read and write heavy; read/write ~ 1/1
    - split files internally in small chunks (4MB)
        benefits:
          on fail the retry will be smaller
          reduce amount of traffic only to the updated chunks
    - keep a local copy of the meta data to save server queries
    - for small changes just upload the diff


4. Capacity Estimation and Constraints
  - total amount of files to keep 100Billion
  - total amount of storage = 100B * 100KB = 10PB

5. High Level Design
  - user will specify a folder on any device.
     Each file there will be uploaded to all the devices.
     The changes in the file will be reflected on every device

 Store files and their metadata
  (name, size, dir, chunks..)
  shared with

we need
  Servers to upload/download files to Cloud Storage and
  Servers to update metadata about files and users.
  way to notify all clients for updates so they can synch

6. Component Design

  a) Client
    - Upload and download files.
    - Detect file changes in the workspace folder.
    - Handle offline and concurrent update conflicts
    - keep a copy of the meta data

    Four components
        1. Internal meta data db - keep all files, chunks, versions and location
        2. Chunker split files and reconstruct them. Detect changed files
        3. Watcher monitors the local workspace and notifies Indexer for distant changes
        4. Indexer
          process events from watcher
          update internal meta data db
          broadcast changes to Synch service

    - client should retry with backoff if server does not respond
    - mobile clients should synch on user demand in order to save battery

  b) Metadata Database
    - responsible for versioning and metadata info abut files, users and workspaces
    - MySQL support ACID operations natively so it will make implementation easier
    - store info about the following objects:
        1. Chunks
        2. Files
        3. User
        4. Devices
        5. Workspace (sync) folders

  c) Synch Service: `SS`
    - the most important part of the system architecture
    - processes file updates made by a client
    - applies changes to all subscribed clients
    - synchronizes clients' local databases with
      the information stored in the remote Metadata DB.
    - Desktop obtain updates from `SS` or send updates to the Cloud Storage and all subscribed
    - clients poll the `SS` for updates after off-line period
    - `SS` checks with the Metadata db for consistency on update request
    - notifies all subscribed users on file update

    Non Functional Requirements
      - transmit min data clients and the Cloud Storage
          + better response time
          + less bandwidth consumption
          + less cloud data for end user
      - needs differencing algorithm
      - divide files into 4MB chunks
      - work only with changed chunks
      - calc SHA-256 for these chunks

    use Communication Middleware (`CM`) between clients and the `SS`
      - `SS` needs scalable synch protocol
      - scalable MQ (message queues)
      - support notifications on change to high number of clients
      - the `SS` instances will receive requests from a global request Queue
      - the `CM` will balance the load

    d) Message Queuing Service (MQS)

      - HA, reliable, scalable
      - asynch message-based communication between clients and the `SS` instances
      - implement two types of queues
          Global Request Queue
            serves as middleware between clients and `SS`
            receives clients request updates on the Metadata DB
            passes them to the `SS` to process them
          Response Queues
            notify clients for updatess
            each client has one
            on delivery delete the message

    e) Cloud/Block Storage
      - Cloud/Block Storage stores chunks of files uploaded by the users.
      - Clients directly interact with the storage to send and receive objects from it.
      - Separation of the metadata from storage enables us to
        use any storage either in cloud or in-house.

8. Data Deduplication
  - Technique used for eliminating duplicate copies of data
      improve storage utilization
      improves bandwidth consumption.
  - Calculate a hash of each new incoming chunk
  - look for present chunk with the same hash in the storage
    if such exist add its reference to the medatada rather than full copy

a) Post-process deduplication
  + clients wait less
  - store duplicate data for short time
  - consume more bandwidth

b) In-line deduplication
  - clients wait for real-time hash calculations
  + optimal storage usage
  + optimal network usage

9. Metadata Partitioning
  a) Vertical Partitioning - user related tables to one server, file...
    Problem: this still does not scale well
    Problem: Joining records between two different servers is hard

  b) Hash-Based Partitioning - hash the File id (unique) for the system
    map the hash to number from 1...256
    Problem: This still leads to unbalanced DB servers
    Solution: Consistent Hashing

10. Caching
  add cache servers 144GB each
    between clients and block servers
    between clients and Metadata DB

11. Load Balancer

  - between Clients and Block servers
  - between clients and meta data servers

12. Permissions and file sharing
  - store permissions in the file object in the metadata DB


* ACID: Atomicity, Consistency, Isolation and Durability
* DAU: Daily Active users
