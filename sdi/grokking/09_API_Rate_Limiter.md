1. What is Rate Limiter?
  Problem: Service receiving huge number of req
  Abstract Solution: Throtling the requests (rate limiting) mechanism
  Solution:
    Rate limiter limits the number of requests
      per entity (user, ip, device)
      for a time window
  Example: An IP can have at most 20 accounts per day

  Rate Limiter (RL) protects from:
    DOS attack
    Brute force pass attempts
    brute force credit card transaction

  RL makes the API more reliable
    we need to limit clients low priority requests
    securitiy: limit the allowed attempts for password
    client device developers may be slopy and request the same data instead of caching it
    to limit the services to human behavior (one req/min, not 1000req/min)
    eliminate spikness in traffic

3. Requirements and Goals of the System
  Functional
    limit the number of req for given time window
    traffic should be distributed on cluster of servers
    return error to user when treshold is crossed
  Non-functional
    HA
    Durable in favor of latency

4. How to do Rate Limiting?
  Define rate and speed at which consumers can access APIs
  Throttling -> controlling the usage of the APIs during a period
    can be defined at App lvl or at Api lvl
    429 ( too many requests error )

5. Different types of throttling

  hard throttling -> put a limit to the number of api requests
  elastic throttling -> allow user to go over the limit if the system has resources

6. Different types of algorithms?

  Fixed window algorithm -> example
    the limit is 1 request/sec
    we have 2 requests in the same sec
    we drop the second request
  rolling window algoritm -> example
    the limit is 4 req per minute
    we have request
    the next minute you are allowed to have at most one request

7. High level design for Rate Limiter
  race conditions for the count (use Redis Lock)
  we need rolling limmit in order not to double the requests per minute

  memory usage
    user id -> 8 bytes (64 bits)
    2 bytes count -> up to 65K(16 bits)
    2 bytes for epoch -> (we cut the miliseconds)
    total -> 12 bytes

    hash table has 20 bytes overhead for each record
    total -> 32 bytes

    we need 4 byte number to lock each user's record
    total -> 36 bytes

  we need to use Redis or Memcache in distributed setup
    the QPS could go up to 10 Million

9. Sliding Window algorithm
  store timesptamp of each request in a Redis Sorted Bag

    1. remove all the timestamps from sorted set older than 1 min
    2. Count the total number of elements
    3. Either throttle the request or save it in the set

  how much memory it will take
    userid 8 bytes
    epoch time 4 bytes
    500 req/hour
    20 bytes overhead for hash-table
    20 bytes overhead for sorted sert
    8 + (4 + 20(sorted set)) * 500 + 20(hash table) = 12 KB or
    (4(epoch time) + 20(sorted set)) * 500 = 12KB

    for 1 million users that is 12GB

11. Data sharding and caching
  based on user id
    for fault tolerance and replication user consistent hashing

  different throttling limits for different APIs than distribute on api

12. Factors on which to rate limit
  IP and User
  before user is logged limit requests to login api using ip
  after the user is logged give him a token and them limit him using it



* DOS -> Denial-of-service = abusive behavior
