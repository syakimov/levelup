1. What is twitter search?
  users can update their statuses whenever they like
  we want to search among user statuses

2. Requirements and Golas of the System
  800 Million DAU
  400 Million status updates
  Average size of a status is 300 bytes
  500 Million status searches
  search is multiple words with 'AND/OR'

3. Capacity Estimation and Constraints
  Storage 400 million new statuses/day * 300 bytes size =>
    400M * 300 = 112GB/day
  Storage/sec => 112GB/day / 86400sec ~ 1.3MB/sec

search(man string api_dev_key,
       man string search_terms,
       man number maximum_results_to_return,
       opt number sort,
       page_token)

?page_token (string): This token will specify a page in the result set that should be returned.

returns JSON
{
  user_id
  user_name
  status_text
  status_id
  creation_time
  likes_count
}

5. High Level Design
  store all the statues in a database
  build an index server keeping track of which word appears in which status
  the index quickly find statuses that users are trying to search

6. Detailed Component Design

  Storage
    112GB * 365days * 5 => 200 TB
    keep storage 80% ful => 240 TB storage
    keep extra copy of the data for fault tollerant system -> 480 TB data (1)
    (1) => we need data partitioning scheme to distribute evenly 240 TB

  A modern server can hold up to 4TB of data => 120 servers for 5 years

  Use MySQL db for storing statuses 
    two cols -> StatusID and StautsText
    partition data based on StatusID
      Problem: create uinque StatusID
      400M new statuses per day in 5 years that is:
      400M * 365 days * 5 years => 730 billion => 5 bytes integer

  Index server
    status queries consist of words
    which words are present in a status

    Size of the index
      we assume we kave 300K English words and 200K nouns => 500K words to index
      we assume word on average has 5 char
      we need 500K * 5 char = 2 500 000 Byte = 2.5MB memory to store all words

    we want to keep index of the statuses for the last 2 years
      730B in 5 years => 292B in 2 years

    How much memory to store all StatusIDs
      Lets assume we need to index a status on 15 words
        -> we need to store each status 15 times

      StatusID = 5 bytes =>  292B * 5 => 1460 GB
      Total storage = (1460 * 15) + 2.5MB ~= 21 TB
      Assuming a high-end server has 144GB of memory,
        we would need 152 such servers to hold our index.
