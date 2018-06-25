**Assumptions**
300M total users
1M daily active users
2M new photos/day, 23 new photos/s
Average photo size 200KB
Keep photos for 5 years
A web server can have 500 concurrent connections
A DB shard is 4TB

1. Instagram is a social networking service
  upload and share pics and videos with oder users

2. Requirements and Goals of the System

Functional Requirements
  - upload/download/view photos.
  - search based on photo/video titles.
  - Users can follow other users.
  - generate and display a user's timeline:
      top photos from all the people the user follows.

Non-functional Requirements
  HA, HR, Eventually consistent
  200ms latency for timeline generation.
  Not in scope
    Adding tags to photos
    searching photos on tags
    commenting on photos
    tagging users to photos
    who to follow

3. Design Considerations
  - read heavy
  - retrieve photos quickly
  - no photo limit: efficient storage is critical
  - low latency while reading images
  - HR - highly reliable service

4. Capacity Estimation and Constraints
  total space required for 1 day = 2M * 200KB = 400GB
  per 5 years 400GB * 365 * 5 = 712 TB

6. Database Schema

Photos {
  PK id:               int
     path:             varchar(256)
     path:             varchar(256)
     latitude:         int
     longtitude:       int
     user_latitude     int
     user_longtitude:  int
     created_at:       datetime
}

Users {
  id          int
  name:       varchar(20)
  email:      varchar(32)
  DOB:        datetime
  created_at: datetime
  last_login: datetime
}

User_follow {
  Composite key
  follower_id: int
  leader_id:   int
}

distributed file storage for photos: HDFS and S3

use distributed key-value store to benefit from NoSQL

table for photo metadata
  `key` would be the `photo_id`
  ‘value’ is object containing `photo_location`, `user_location`, `creation_timestamp`

use Cassandra db for tables:
  (`user_photo`) relationships between users and photos
  (`user_follows`) list of people a user follows

table `user_photo`
  key: `user_id`
  value: list of `photo_ids` stored in different columns

7. Component Design
  Problem: Writes can consume all connections which creates bottleneck

  Solution: Separate reads from writes

8. Reliability and Redundancy
  Problem we want redundancy - when a server dies we want to have the data

  Create replicas to ensure there is no single point of failures

9. Data Sharding
  a) Partitioning based on user id
  `user_id` % `number_of_shards` = the shard id where the user photo reside
    Problems:
      Some power users will make uneven distribution
      How to split user data into two shards?
      Handle high latency due to different shards distribution
      How to generate photo ids
        Solution: append incremental sequence to the user id
  b) Partitioning based on photo id
    Problem: generate unique sequence for photo ids
    Solution I:
      Create a `photo id table` with one 64 bit incremental
      whenever you want to create a photo add a new record in this db
      Problems:
        Single point of failure
        Solution: Place a load balancer and use two sequences: odd/even
    Solution II: use KGS

How can we plan for future growth of our system?

We can have a large number of logical partitions to accommodate future data growth.
In the beginning, multiple logical partitions reside on a single physical database server.

Since each database server can have multiple database instances on it,
  we can have separate databases for each logical partition on any server.
  Migrate logical partitions from overflowing server to another.

  We can maintain a config file (or a separate database) in order to
  map our logical partitions to database servers
    this will enable us to move partitions around easily.

10. Ranking and Timeline Generation
  Pre-generate timeline
    dedicated servers generating timeline for users
    when user wants to see the timeline
      render what is in `user_timeline` table
      add new photos on the fly
      Problem: how to update user timeline while he is browsing
        Solution I: Pull
          Problem: Many Pulls will be useless and there will be latency
        Solution II: Push
          implement Long Poll, server sends only when there is new data
          Problem: Server has to push to many clients when a celebrity adds photo
        Solution III: Hybrid
          Push for regular and pull for celebrities

11. Timeline Creation with Sharded Data
  Problem: to create timeline we need to index on created at
  Solution: Integrate epoch time in photo id -> append unique sequence to epoch time

12. Cache and Load balancing
  Problem: To serve globally distributed users,
           our service needs a massive-scale photo delivery system.

  Solution:
    - Push content closer to the user using a large number of
      geographically distributed photo cache servers and use CDNs
    - Use cache with LRU eviction policy for metadata servers
      to cache hot database rows.

* Block storage, ref: https://cloudian.com/blog/object-storage-vs-block-storage/
* HDFS and S3
* wide-column database - Cassandra
* key-value stores maintatin certain number of replicas offer reliability
* Long Poll
* Designing Facebook news feed
* Data sharding in designing Twitter
