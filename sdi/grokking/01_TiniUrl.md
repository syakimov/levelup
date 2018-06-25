1. Why we need URL shortening?
  Problem shorten long url-s
  when user type the short url he is redirected to the long one
  usefull for tweets where there is text limit

2. Requirements and Goals of the System
  - clarify requirements at the beginning
  - find the exact scope of the system

  2.1 Functional Requirements:
    - generate a shorter and unique alias (short link) of logn URL
    - short link redirects to long link
    - pick a custom short link
    - SL have expiration which can be set by users

  2.2 Non-Functional Requirements:
    - HA
    - real time with min latency
    - SL should not be guessable

  2.3. Extended Requirements:
    - Analytics (redirection count)
    - Rest api

3. Capacity Estimation and Constraints
  - Read heavy (1:100) (**Assumed**)

  3.1 QPS, Reads/s Writes/s
    write_requests_per_month = 500Million
    => read_requests_per_month = 50Bbillion

    seconds_per_month = (30 days * 24 hours * 3600 seconds)

    write_requests_per_month / seconds_per_month = 200/s
    read_requests_per_month / seconds_per_month = 19 000/s

  3.2 Storage
    - store URL and SL

      max_expiration = 60 months (**Assumed**)
      max_obj_count = write_requests_per_month(500M) * max_expiration = 30Bilions URL objects
      record_size = 500 bytes (**Assumed**)
      needed_storage = record_size * max_obj_count = 15 TB

  3.3 Bandwidth
    - incoming data = object size * write req/s
        => 200 URL/s * 500 bytes = 10 KB/s

    - outgoing data = object size * read req/s
        => 19 000 * 500 bytes = 9 MB

  3.4 Memory
    (**Assume** Pareto 80/20)

    total reads per day = total_reads_per_month / 30 = 50B / 30 =  1.7 Billion
    cache 20% of the reads for a day
      => 500KB * total reads per day * 20% = 170GB

  3.5 Conclusion
    with Assumptions 500Milion new URL and 100:1 read/write

    - Writes - 200/s
    - Reads - 19000/s
    - Incoming data 100KB/s
    - Outgoing data 9MB/s
    - Storage for 5 years 15TB
    - Memory for cache 170GB

4. System APIs

  Create Url {
    creatURL(api_dev_key,
            original_url,
            custom_alias=None,
            user_name=None,
            expire_date=None)

    Parameters:
      api_dev_key: The API developer key of a registered account. This will be used to, among other things, throttle users based on their allocated quota.
      user_name: Optional user name to be used in encoding.
      expire_date: Optional expiration date for the shortened URL.

    Returns:
      A successful insertion returns the shortened URL
      A failed insertion returns an error code.
  }

  Delete Url {
    deleteURL(api_dev_key, url_key)
    url_key: Short Link

    Returns 200 OK
  }

  Protections:
    limit users read/write via api_dev_key

5. Database Design
  Facts:
    - 30 Billion objects
    - object size < 1K (500B) small
    - only relationship btw objects is user -> URL
    - Read heavy

  DB schema {
    URL_table
      PK: Hash:         varchar(16)
          original_url: varchar(512)
          created_at:   datetime
          expiration:   datetime
      FK: user_id:     int

    User_table
      PK: user_id: int
          name:       varchar(20)
          email:      varchar(32)
          created_at: datetime
          last_login: datetime
  }

  DB choise Dynamo or Cassandra
    - no relations between objects
    - 30 billion rows easier to scale

6. Basic System Design and Algorithm
    Problems:
      - generate a short and unique key for a given URL?
      - each time the same input URL will result in diff SL

    a. Encoding actual URL
      - we need at least 30B uniq strings
      - we need some uniqueness factor when encoding (user_id, ever increasing seq)
        => this will result in performance issues
    b. Generate keys offline
      - Key Generation Servise (KGS) (simplify and speed-up)
          - generate random six letter strings beforehand
          - store them in DB

      Concurrency Problems - two servers try to access same key
      Solution ->
        KGS keeps keys in two tables
          used_keys_table
          not_used_keys_table
        KGS moves keys from one to another and keeps them in memory for faster access
          KGS needs to make the sync
            - after loading in memory move keys to used_table
            - use thread lock data structure for keys

      DB size => 6char * 68K uniq keys = 412 GB

      KGS is single point of failure so we make replica

      Each app server can cache some of the lost keys on fail would not be a prob

7. Data Partitioning and Replication
    Problem: divide and store data on many DB servers

      a. Range Based Partitioning
          => based on starting letter store them in diff servers, combine for
            not common letters. This leads to unbalansed servers.
      b. Hash-Based Partitioning: determine to which server the data goes based
          => on the hash
      - solve overloading partitions using Consistent Hashing

8. Cache
    Problem: Use Memcache to cache most visited urls in the app servers

    - Size: 170 GB memory to cache 20% so a machine with 250GB will be fine
    - cache eviction policy: Least Recently Used (LRU)
    - Linked Hash Map
    - replicate cache to distribute load
    - cache invalidation

9. Load Balancer
  - Between Clients and the App servers
  - Between App servers and DB servers
  - Between App servers and Cache servers
10. Purging or DB cleanup
  - remove link when user tries to access it and it is already expired
  - run light weight service or cron job when traffic is low
  - return expired key in the db for next usage

12. Security and Permissions
  Problem: users create private URLs available for set of users

  - store permission lvl with each URL in the db
  - store user ids which can access certain link
  - key - the hash, cols the user ids

* Thousand -> Million -> Billion (T-> M-> B)
* Queries Per Second (QPS)
* Memory table
  Bit = 1/0
  Byte = 8 bits = 1 char
  Kilobyte(KB) = 1,024 bytes (2-3 paragraphs of text)
  Megabyte(MB) = 1,024 kilobytes (900 pages) (5MB = mp3 song) (1.6MB web page)
  Gigabyte(GB) = 1,024 megabytes (650 webpages)
  Terabyte(TB) = 1,024 gigabytes
  Petabyte(PB) = 1,024 terabytes
  ref: https://www.computerhope.com/issues/chspace.htm
* SOAP vs REST ref: https://stackify.com/soap-vs-rest/
* Consistent Hashing
