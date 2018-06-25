*Assumptions*
(1) 5B searches/day
(2) From 1 => 60K searches/sec

1. What is Typehead suggestion?

  Search fro known and frequently searched terms
  predict the query based on the chars the user has entered and
    give a list of suggestions
  guiding the user

2. Requirements and Goals
  as the user types the system should suggest top 10 erms
  real time
  200ms latency

3. Basic system design
  use trie
  case insensitive
  
  how to find top suggestions
    store the count of solutions that terminated at each node

  store top suggestions with each node
    speed up solution -> required
    increase memory usage
    each parent calls his children to count the occurence

  update the trie
    not on every request
    update it offline
      log the requests -> every 1000th searched term
      take the current snapshot of the trie and update it with new terms

      1. make a copy of the trie on each server to update it offline
      2. ms config. Update the slave while the master is serving traffic.
        Slave becomes master and serves traffic
        update old master and let him serve traffic

    update frequencies of suggestions
      we store frequencies with each node
      only update the diff
      substract the frequencies based on Exponential Moving Average
      give more weight on latest data

  how to remoeve a term from the trie
    add a filter layer on each server

  criteria for suggestions
    freshness, user location, language, demographics, personal history

5. Scale Estimation
  only 20% of the queries will be unique
  inddex only 50% of the searched terms
  assume we have 100M unique terms
  100M * 30 bytes -> 3GB per day

  storage
    3 words -> 15 char -> 30 bytes

6. Data Partition

* Map-Reduce (MR) setup
* EMA -> Exponential Moving Average
