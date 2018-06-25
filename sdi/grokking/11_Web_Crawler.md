** Web Crawler

  Browses the WWW
  collects documents by recursively fetching links
    from a set of starting pages
  download all the pages to create an index on them to perform faster searches

  Secondary uses
    test web pages and links for valid syntax and structure
    monitor sites to see when their structure or contents change
    maintain mirror sites for popular Web sites
    search for copyright infringements

2. Requirements and Goals of the System

  Scalability:
    Our service needs to be scalable such that it can crawl the entire Web
  Extensibility:
    New functionality will be added in time

3. Design
  What types of media are we targeting?
    `general-purpose` crawler download different media types
      break down the parsing module into different sets of modules:
        HTML, images, videos
    We target HTML crawler

  What types of protocols are we targeting?
    For now only HTTP
    FTP is easily extendable

  What is the expected number of pages we will crawl?
    One Billion websites
  How big will the URL database become?
    15 billion different web pages

  RobotsExclusion
    download robot.txt before searching the page

4. Capacity Estimation and Constraints
  Fetch pages per second - 6200 pages/sec
  average HTML page size is 100KB + 500 bytes metadata
    15B * (100KB + 500 bytes)  ~ 1.5 petabytes
    with 70% full db -> 2.14 petabytes

5. High level design
  Web crawler take a list of seed URLs as input and execute the algorithm

  1. Pick a URL from the unvisited URL list.
  2. Determine the IP Address of its host-name.
  3. Establishing a connection to the host to download the corresponding document.
  4. Parse the document contents to look for new URLs.
  5. Add the new URLs to the list of unvisited URLs.
  6. Process the downloaded document, e.g., store it or index its contents
  7. Go back to step 1

6. BFS or DFS
  Usually BFS but DFS reduces the handshakes overhead
  Path-ascending crawling -> technique to find not mentioned urls
    http://foo.com/a/b/page.html
    http://foo.com/a/b
    http://foo.com/a

  Efficient WC
    - Prioritize which web pages to download
    - Cope with changes -> the site might have changed by the end of downloading

  Componnents
    - URL frontier: To store the list of URLs to download and also prioritize which URLs should be crawled first.
    - HTTP Fetcher: To retrieve a web page from the server.
    - Extractor: To extract links from HTML documents.
    - Duplicate Eliminator: To make sure same content is not extracted twice unintentionally.
    - Datastore: To store retrieve pages and URL and other metadata.
