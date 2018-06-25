*Assumptions*
  (1) 1.5B total users
  (2) 800M DAU
  (3) A user views 5 videos per day => 800M * 5 / 86 400 = 46K videos/sec
  (4) Writes/reads = 1/200
  (5) From (3) and (4) => 46k/200 = 2300videos/sec
  (6) 1 min video = 50MB storage
  (7) write 500 hours videos/min
  (8) storage/min = From (5) and (6) => 500 hours * 60 min * 50MB = 1500GB/min = 25GB/sec
  (9) A video upload takes 10MB/min bandwidth
  (10) From (6) and (8) 500 hours * 60 min * 10MB => 300GB/min = 5GB/sec incoming data
  (11) From (4) and (10) => 5GB/s * 200 = 1TB/s outgoing data
  (12) Thumbnail is 5KB

1. Youtube
  vide sharing website
  upload/view videos
  share
  rate
  report and comment

2. Requirements and Goals

Functional Requirements:
  upload videos
  share and view videos
  search based on video titles
  record stats of videos, e.g., likes/dislikes, total number of views, etc.
  add and view comments on videos.

Non-Functional Requirements:
  HR
  HA in favor of Consistency
  time experience video stream

Not in scope:
  Video recommendation, most popular videos, channels, and subscriptions, watch later, favorites, etc.

3. Capacity Estimation and Constraints
  check *Assumptions*

4. System APIs

uploadVideo(string `api_dev_key`,         # API dev key of registered acc
            string video_title,
            string vide_description=None,
            string[] tags=None,
            string category_id,           # Film, song, people
            string default_language,
            string recording_details,     # Location
            stream video_contents)

Returns HTTP (202) request accepted
User receives delayed email with link to the video
Expose API for checking the video status

searchVideo(string `api_dev_key`,
            string search_query,
            string user_location=None,
            number max_videos_to_return,
            string page_token)

Returns
{
  0: { `video_title`, thumbnail, `created_at`, `views_count` },
  1: ...
}

5. High Level Design

5.1 Processing Queue - each uploaded video goes through
      encoding
      thumbnail generation
      lastly stored
5.2 Encoder - encode into multiple formats
5.3
5.4 Video and Thumbnail storage -> store them in distributed file storage
5.5 User DB
5.6 Video metadata storage -> { title, file path, user, total views, (dis)likes, comments }

6. Database Schema

Video metadata storage - MySql
  videos
    id
    title
    description
    size
    thumbnail
    user_id
    likes
    dislikes
    views
    created_at

  comment
    id
    video_id
    user_id
    comment
    created_at

  users
    id, name, email, address, age, registered_at, last_login

7. Detailed Component Design
    Read heavy service

    video storage -> HDFS or GlusterFS

    How to efficiently manage read traffic
      since we have copies of a video on diff servers
        distribute read traffic on diff servers
      for metadata use master-slave config
        master handles uploads
        slaves handle reads
        this will be a hit on Consistency
          since the video will not be available right after it is uploaded
          this is ok in favour of HA

    thumbnails handling
      each thumbnail is 5KB
      efficiently store thumbnails -> each video has 5 thumbnails
      we need fast read -> users can whach one video but 20 thumbnails
      use Bigtable
        as it combines multiple files into one block
        very efficient for reading small amounts of data
      cache some of the thumbnails

8. Metadata Sharding
  based on user id
    hot users will be bottleneck
    users do not store equal amount of videos
  based on video id
    chose random server and store video metadata
    what about hot videos -> place a cache before db servers

9. Video Deduplication
    duplicate videos lead to
      wasted storage
      wasted cache
      newtwork usage

    from user perspective
      inefficient search
      longer video startup time

    we have to check inline if a video is being duplicated
      this is done with matching algorithms (Block Matching, Phase Correlation)
      if it is then use the saved copy and stop the upload
      if it is with higher quality delete the old one
      if it is part of existing one divide it and use it

10. Load Balancing
  use consistent hashing to balance the load between cache servers
  to resolve popular video bottleneck
    a busy server can redirect to less busy in the same cache location (dynamic http redir)
    keep in mind that each http redirect has http request overhead

11. Cache

  push content closer to  the user
    using large number of geographically distributed video cache servers
  cache metadata serverse

12. content Delivery Network (CDN)
  System of distributed servers
    that deliver well content to user
      based on the geographic location of the user

  most popular videos to CDNs
    replicate content in multiple places
    use cache a lot and serve video mainly from memory
  less popular videos can be served from system servers

13. Fault Tolerance
  use Consistent Hashing for distribution among db servers
    helps in replacing dead server
    distribute load among servers

* HDFS
* GlusterFS
* Bigtable
* CDN - Content Delivery Network
* Dynamic HTTP redirections
