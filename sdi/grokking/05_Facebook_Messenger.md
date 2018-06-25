*Assumptions*

(1) 500M daily acttive users
(2) Each user sends 50 msges/day => 20B msges/day in total
(3) A text msg is 100 Bytes
(4) we store each msg for 5 years
(5) A modern server an hande 50K concurrent connections
(6) One DB server is 4TB

1. What is FB Msger?
   - text-based instant msg service to users
   - users can chat from mobile and web apps

2. Requirements & Goals

  Functional Requirements:
    - support one-on-one conversations between users
    - support persistent storage of chat history
    - keep track of online/offline statuses of its users
  Non-functional Requirements:
    - real-time, min latency
    - consistent (chat history the same in on devices)
    - HA is desirable, will be sacrefised in order of consistency
  Extended Requirements:
    - group chat
    - push notification on new message

3. Capacity Estimation and Constraints

Storage
  storage per day = From (2) and (3) => 20M * 100KB = 2TB/day storage (A)
  total storage = From (A) => 365 * 5 * 2TB = 3.6PB storage

Bandwidth
  86400 seconds in a day (3600*24)
  outgoing  data per second = From (A) storage per day / secs in day = 2TB / 86400 = 25MB/s
  incoming is the same as outgoing because the msg is received by us

4. High Level Design
  the msg sender will chat through a chat server
  the chat server will save the message

  4.1 Msg Handling
    Pull model
      Problems
        co much wasted requests - not efficient bandwidth consumption
        we want min latency
        server needs to keep track of the pending msges
    Push model
      overview
        each client keeps open connection with the server
        server sends the msg immediatelly on the connection
      implementation
        Long Polling or WebScket
        server keeps hash table user_id to open connection
        server needs to store pending msges until users go online

      how many servers we need
        one conn for each user -> 500M users
        from (5) 500M / 50K -> 1K servers
      how to know which server keeps the connection to the user
        in front of the server we need a load balancer to map the user id to server
      how to order the messages
        keep a sequence number with every message for each client

  4.2 Store and retrieve msges from DB

    how to save in db
      Start a separate thread, which will work with the database to store the message
      Send an asynchronous request to the database to store the message

    DB requirements
      efficiently work with db connection pool
      retry failed requests?
      Where to log requests that failed even after certain retries?
      retry logged requests when issues are resolved?

    DB options
      we need low latency because we read/write small msges all the time
      we need high rate of small updates and quick fetch of updates
      we need db that stores efficiently vaiable size of data
      MySQL and MongoDB are not an option -> high latency
      HBase meets the requirements

  4.3 Manage user's status
    Optimization:
      Whenever a client starts the app, it can pull current status of all firends
      Whenever a user sends a message to another user that has gone offline update status
      Server broadcasts when user goes online with delay (go offline immediately)
      User can pull status for viewpoint of uses not often
      pull the status of a friend when the user starts chat with him

5. Data partitioning
    based on user id - use hash of the user id to alocate to db partition
      fetching history for one user will be fast
    based on msg id will be slow for history fetch

6. Cache
  cache last 15 msges for the last 5 user conversation
  since we store all users' msges on one shard we place cache on that machine

7. Load Balancer
    Place a LB in front of the chat servers which has the map user id -> chat server
    Place a LB in front of the cache servers which has the map user id -> cache server

8. Failure and replication
    when a chat server fails make all clients reconnect automatically
    store multiple copies of the user data and messages on different servers
    Reed-Solomon encoding to distribute and replicate

9. Group chat
  Group chat objects in the system
    Store group-chat objects on the chat servers.
    GC obj have unique id and have list of user ids

  Flow
    LB direct each group chat msg based on id to the responsible chat server
    chat server iterate through all users to find the chat servers their connections reside
    the chat server sends the msg through his peer connections

  Storage
    store all group chat messages on diff table partinioned on their id

10. Push notifications
    Enable sending msges to offline users
    we have to use manifacture's push notification server (MPNS)
    Notification server takes the messages for offline users and sen them to (MPNS)

* HBase DB: column-oriented key-value NoSQL database that
  can store multiple values against one key into multiple columns
* Google BigTAble
* HDFS - Hadoop Distributed File System
* Reed-Solomon distribution and replication
