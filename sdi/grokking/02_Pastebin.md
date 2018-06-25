1. Pastebin -> users store plain text and they are given url to access it.
Assumptions
  - read vs write requests = 5/1
  - 1Million pastes per day
  - 5Million reads per day
  - paste object is 10KB (5 pages)
  - store texts for 10 years

2. Requirements and Goals of the System

Functional Requirements:
  - upload or "paste" text and get a unique URL to access it
  - expiration as option
  - custom alias for the paste

Non-Functional Requirements:
  - HA, uploaded date should not be lost
  - Min latency
  - paste links should not be guessable

Extended Requirements:
  - Accessed paste count
  - Build REST/SOAP API

3. Design Considerations

- Impose 10MB text limit for users
- Impose size limit on custom URLs

4. Capacity Estimation and Constraints

Traffic
  - pastes per sec = 1 000 000 / (24h * 3600 sec) = 12 paste/sec
  -  reads per sec = 5 000 000 / (24h * 3600 sec) = 58 reads/sec

Storage
  - 1Million per day * 30 days * 12months * 10 years * 10KB  = 36TB total storage
  - there will be 3.6Billion Pastes in 10 years so we need to generate keys
  - base64 encoding need six letter strings and produce ~70 billion unique strings

Bandwidth
  incoming data - 12 pastes/sec * 10KB size = 120 KB/s
  outgoing data - 60 pastes/sec * 10KB size = 0.6 MB/s

Memory
  Cache 20% of read requests per day
  20% * 5Million * 10KB = 10GB

5. System API

Add Paste {
  addPaste(api_dev_key,
          paste_data,
          custom_url=None,
          user_name=None,
          paste_name=None,
          expire_date=None)

  Parameters:
  `api_dev_key`: used to throttle requests over the limit
  `paste_data`: Textual data of the paste.
  `custom_url`: Optional custom URL.
  `user_name`: Optional user name to be used to generate URL.
  `paste_name`: Optional name of the paste
  `expire_date`: Optional expiration date for the paste.

  Returns:
  A successful insertion returns the URL otherwise, returns an error code.
}

Get Paste {
  getPaste(api_dev_key, api_paste_key)
  return paste or not found
}

Delete Paste {
  deletePaste(api_dev_key, api_paste_key)
  return true of false
}

6. DB Design
  - 3.6 Billions of records
  - meta data per object is going to be small 100 bytes
  - Paste object can be a few MB (medium size)
  - No relations between objects
  - read heavy service

  DB Schema
  `Pastes_table`
    PK Hash:          varchar(16)
      Url:           varchar(512)
      content_path:  varchar(512)
      exp_date:      datetime
      user_id:       int
      creation_date: datetime

  `Users_table`
    PK id:         int
    name:          varchar(20)
    email:         varchar(32)
    creation_date: datetime
    last_login:    datetime

7. High Level Design
  segregate storage layer with one database storing metadata
  while the other storing paste contents in some sort of block storage or a database.

8. Component Design

App layer
  Write request
    a) Generate unique key on the fly and retry on duplicate key error in the db
    b) use KGS that generates all six number base64 strings beforehand
      store them in two tables -> used and not_used
      cache some in memory and move them in used
      create a KGS replica to avoid single point of failure
    each app server can cache some of the keys as failure is acceptable due to many keys
  Read request
    app servers query the db servers

* Block storage
