*Assumptions*
(1) 1B total users
(2) 200M daily active users (DAU)
(3) 100M new tweets 
(4) User follows 200 people on average
(5) User favorites 5 tweets on average per day
(6) From (2) and (5) => 1B favorites
(7) Reads per day = 28B/day
(8) A photo is 200KB, 20% of the tweets has it
(9) A video is 2MB, 10% of the tweets has it


1. What is Twitter?
  users can write/read short msges 140 char long (tweets)
  guests can only read tweets
  user can access the platform through web/mobile and SMS interface

2. Requirements and Goals

Functional Requirements
  Users should be able to post new tweets.
  A user should be able to follow other users.
  Users should be able to mark tweets favorite.
  Tweets can contain photos and videos.
  The service should be able to create and display user's timeline
    consisting of top tweets from all the people the user follows.

Non-functional Requirements
  HA
  Acceptable latency of the system is 200ms for timeline generation.
  Eventual consistent in favour of HA

Extended Requirements
  Searching tweets.
  Reply to a tweet.
  Trending topics – current hot topics/searches.
  Tagging other users.
  Tweet Notification.
  Who to follow? Suggestions?
  Moments.

3. Capacity Estimation and Constraints
  storage
    each tweet has 140 char -> 280 bytes and Meta data is 30 bytes = 310 (A)
    total storage needed for tweets per day = from (3) and (A) => 100M * bytes = 30GB/day
    media storage = From (8) and (9) => 24TB/day
  bandwidth
    24TB/day => 2900MB/sec
    show photo on every tweet but every third view whatches video, so outgoing is
    35GB/s
  
4. System APIs
  tweet(string api_dev_key,
        string tweet_data,
        string tweet_location=None,
        string user_location=None,
        number[] media_ids,
        maximum_results_to_return)

  `api_dev_key`: The API developer key of a registered account.
                 Used to throttle users based on their allocated quota.
  `tweet_data`: The text of the tweet, typically up to 140 characters.
  `tweet_location`: Optional location (longitude, latitude) this Tweet refers to.
  `user_location`: Optional location (longitude, latitude) of the user adding the tweet.
  `media_ids` (): Optional list of `media_ids` to be associated with the Tweet.

Returns: success: tweet url; failure: error code

5. High Level System Design
    new tweets per second: 100M (3) / 86 400 = 1150tweets/s
    reads per second: 28B (7) / 86 400 = 325 000 tweets/s
    this traffic has peeks and downs

    components
      LB in front ot app servers
      app servers
      db that support efficient reads
      file storage for video and media

6. Database schema
tweets
 PK id:              int
    user_id:         int
    content:         varchar(140)
    latitute:        int
    longtitute:      int
    user_latitute:   int
    user_longtitute: int
    created_at:      datetime
    favorites_count: int

users
 PK id:         int
    name:       varchar(20)
    email:      varchar(32)
    dob:        datetime
    created_at: datetime
    last_login: datetime

`users_follow`
  `leader_id`:   int
  `follower_id`: int

favorites
 PK id:         int
    tweet_id:   int
    user_id:    int
    created_at: datetime

7. Data Sharding

  based on `user_id`
    Problem: this will lead to uneven distribution due to hot users
    Solution use consistent hashing or reparation/redistribution
  based on `tweet_id`
    in order to create the user timeline we need to:
      find all the users the client follows
      query all the db servers for their last tweets
      merge the answer
  based on tweets `created_at`
    this will help us index the most recent ones easy
    but will also lead to unevent read/write distribution
  combine `created_at` with unique sequence
    we store tweet creation in the key
    we need to make tweet id universally uniq (db servers odd/even seq)
    reset auto incrementing sequence every second
    append seq id to epoch time
    we need 31 bits for epoch time (A)
    we need to store 100K/s so 2^17(130K) will be enough => 17bits for seq (B)
    From (A) and (B) => tweet_id will be 64bits for the next 100 years

8. Cache
  place aggregation server between db and app servers
    the aggr. server will query the cache, db and file servers and will return data
    we can use a data structure: hash with key <int>user_id -> <tweets> double linked list
    insert new tweets at the head of the linked list and remove from the back

9. Timeline Generation check FB newsfeed

10. Replication and Fault Tolerance
  replicate the db servers and use the replicas only for reads
  this will give fault tolerance and better writes

11. Load Balancing
  between clients and app servers
  between app servers and db replication servers
  between aggregation servers and cache servers

12. Monitoring
  tweets per day, daily tweet peek
  timeline delivery stats
  average latency after refresh

13. Extended requirements
  how to serve tweets by the client
    get all the latest tweets from following and sort them by date
    paginate them on clients viewport
    cache next tweets to speed refresh

  retweet -> store parent tweet id

  Trending topics
    cache most frequentl hashtags/searched queries
    update them on interval
    rank topics on likes and views

  who to follow
    query the followers and friends of friends
    give preference to people with more followers

* REST vs SOAP APIs
* composite key
* Designing Instagram
* epoch time
* FB newsfeed
* Design Twitter Search
