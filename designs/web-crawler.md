# Web Crawler

## 1.Summary
![overview](../img/web-crawler-overview.png)
![detail](../img/web-crawler-detail.png)

## 2. Requirements and Goals of the System

Let’s assume we need to crawl all the web.
**Scalability:** Our service needs to be scalable such that it can crawl the entire Web and can be used to fetch hundreds of millions of Web documents.
**Extensibility:** Our service should be designed in a modular way with the expectation that new functionality will be added to it. There could be newer document types that needs to be downloaded and processed in the future.

## 3. Some Design Considerations

**1. Is it a crawler for HTML pages only? Or should we fetch and store other types of media, such as sound files, images, videos, etc.?**
This is important because the answer can change the design. If we are writing a general-purpose crawler to download different media types, we might want to break down the parsing module into different sets of modules: one for HTML, another for images, or another for videos, where each module extracts what is considered interesting for that media type.

**2. What protocols are we looking at? HTTP? What about FTP links? What different protocols should our crawler handle?**
For the sake of the exercise, we will assume HTTP. Again, it shouldn’t be hard to extend the design to use FTP and other protocols later.

**3. What is the expected number of pages we will crawl? How big will the URL database become?**
Let’s assume we need to crawl one billion websites. Since a website can contain many, many URLs, let’s assume an upper bound of 15 billion different web pages that will be reached by our crawler.

**4. What is ‘RobotsExclusion’ and how should we deal with it?**
Courteous Web crawlers implement the Robots Exclusion Protocol, which allows Webmasters to declare parts of their sites off limits to crawlers. The Robots Exclusion Protocol requires a Web crawler to fetch a special document called `robot.txt` which contains these declarations from a Web site before downloading any real content from it

## 4. Capacity Estimation and Constraints

If we want to crawl 15 billion pages within four weeks, how many pages do we need to fetch per second?

`15B / (4 weeks * 7 days * 86400 sec) ~= 6200 pages/sec`

**What about storage?** 
Page sizes vary a lot, but, as mentioned above since, we will be dealing with HTML text only, let’s assume an average page size of 100KB. With each page, if we are storing 500 bytes of metadata, total storage we would need:

`15B * (100KB + 500) ~= 1.5 petabytes`

Assuming a 70% capacity model (we don’t want to go above 70% of the total capacity of our storage system), total storage we will need:

`1.5 petabytes / 0.7 ~= 2.14 petabytes`

## 5. High Level design
The basic algorithm executed by any Web crawler is to take a list of seed URLs as its input and repeatedly execute the following steps.
1. Pick a URL from the unvisited URL list.
2. Determine the IP Address of its host-name.
3. Establish a connection to the host to download the corresponding document.
4. Parse the document contents to look for new URLs.
5. Add the new URLs to the list of unvisited URLs.
6. Process the downloaded document, e.g., store it or index its contents, etc.
7. Go back to step 1 

### How to Crawl? Breadth first or depth first?
- Breadth-first search (BFS) is usually used ( sake of politeness )
- Depth First Search (DFS) is also utilized in some situations, such as, if your crawler has already established a connection with the website, it might just DFS all the URLs within this website to save some handshaking overhead

### Path-ascending crawling: 
Path-ascending crawling can help discover a lot of isolated resources or resources for which no inbound link would have been found in regular crawling of a particular Web site. In this scheme, a crawler would ascend to every path in each URL that it intends to crawl. 
For example, when given a seed URL of http://foo.com/a/b/page.html, it will attempt to crawl /a/b/, /a/, and /.

### Difficulties in implementing efficient web crawler
1. Large volume of Web pages
2. Rate of change on web pages

### Components

1. **URL frontier:** To store the list of URLs to download and also prioritize which URLs should becrawled first.
2. **HTTP Fetcher:** To retrieve a web page from the server.
3. **Extractor:** To extract links from HTML documents.
4. **Duplicate Eliminator:** To make sure the same content is not extracted twice unintentionally.
5. **Datastore:** To store retrieved pages, URLs, and other metadata

![Screen Shot 2021-10-03 at 6 08 11 PM](https://user-images.githubusercontent.com/1195878/135758726-d6fb42b2-82b9-47b8-bb15-48d5f0e798d4.png)

## 6. Detailed Component Design

![Screen Shot 2021-10-03 at 6 10 30 PM](https://user-images.githubusercontent.com/1195878/135758840-a18cd408-bcb5-411e-95d7-aaccefa5141a.png)

### 1. The URL frontier: 
The URL frontier is the data structure that contains all the URLs that remain to be downloaded. We can crawl by performing a breadth-first traversal of the Web, starting from the pages in the seed set. Such traversals are easily implemented by using a FIFO queue. Since we’ll be having a huge list of URLs to crawl, we can distribute our URL frontier into multiple servers. Let’s assume on each server we have multiple worker threads performing the crawling tasks. 
Let’s also assume that our hash function maps each URL to a server which will be responsible for crawling it.

Following politeness requirements must be kept in mind while designing a distributed URL frontier:
1. Our crawler should not overload a server by downloading a lot of pages from it.
2. We should not have multiple machines connecting a web server.

### 2. The fetcher module: 
The purpose of a fetcher module is to download the document corresponding to a given URL using the appropriate network protocol like HTTP. 
As discussed above, webmasters create robot.txt to make certain parts of their websites off limits for the crawler. To avoid downloading
this file on every request, our crawler’s HTTP protocol module can maintain a fixed-sized cache mapping host-names to their robot’s exclusion rules.

### 3. Document input stream: 
Our crawler’s design enables the same document to be processed by multiple processing modules. To avoid downloading a document multiple times, we cache the
document locally using an abstraction called a Document Input Stream (DIS).

A DIS is an input stream that caches the entire contents of the document read from the internet. It also provides methods to re-read the document. The DIS can cache small documents (64 KB or less) entirely in memory, while larger documents can be temporarily written to a backing file.
Each worker thread has an associated DIS, which it reuses from document to document. After extracting a URL from the frontier, the worker passes that URL to the relevant protocol module, which initializes the DIS from a network connection to contain the document’s contents. The worker then passes the DIS to all relevant processing modules.

### 4. Document Dedupe test:
To perform this test, we can calculate a 64-bit checksum of every processed document and store it in a database. For every new document, we can compare its checksum to all the previously calculated checksums to see the document has been seen before. We can use MD5 or SHA to calculate these checksums.

We use cache with LRU and if not found will go through the database **(for 15B URLs it will be about 120GB)**

### 5. URL filters: 
The URL filtering mechanism provides a customizable way to control the set of URLs that are downloaded. This is used to blacklist websites so that our crawler can ignore them. Before adding each URL to the frontier, the worker thread consults the user-supplied URL filter. We can define filters to restrict URLs by domain, prefix, or protocol type.

### 6. Domain name resolution: 
Before contacting a Web server, a Web crawler must use the Domain Name Service (DNS) to map the Web server’s hostname into an IP address. DNS name resolution will
be a big bottleneck of our crawlers given the amount of URLs we will be working with. To avoid repeated requests, we can start caching DNS results by building our **local DNS server.**

### 7. URL dedupe test: 
While extracting links, any Web crawler will encounter multiple links to the same document. To avoid downloading and processing a document multiple times, a URL dedupe test must be performed on each extracted link before adding it to the URL frontier.
To perform the URL dedupe test, we can store all the URLs seen by our crawler in canonical form in a database. To save space, we do not store the textual representation of each URL in the URL set, but rather a fixed-sized checksum.


## 7. Fault tolerance
We should use consistent hashing for distribution among crawling servers. Consistent hashing will not only help in replacing a dead host, but also help in distributing load among crawling servers.
All our crawling servers will be performing regular checkpointing and storing their FIFO queues to disks. If a server goes down, we can replace it. Meanwhile, consistent hashing should shift the load to
other servers.

## 8. Data Partitioning
Our crawler will be dealing with three kinds of data: 
1) URLs to visit 
2) URL checksums for dedupe 
3) Document checksums for dedupe.

Since we are distributing URLs based on the hostnames, we can store these data on the same host. So, each host will store its set of URLs that need to be visited, checksums of all the previously visited URLs and checksums of all the downloaded documents. 
Since we will be using consistent hashing, we can assume that URLs will be redistributed from overloaded hosts.
Each host will perform checkpointing periodically and dump a snapshot of all the data it is holding onto a remote server. 
This will ensure that if a server dies down another server can replace it by taking its data from the last snapshot.

## 9. Crawler Traps
There are many crawler traps, spam sites, and cloaked content. A crawler trap is a URL or set of URLs that cause a crawler to crawl indefinitely. Some crawler traps are unintentional. For example, a symbolic link within a file system can create a cycle. 
Other crawler traps are introduced intentionally.
For example, people have written traps that dynamically generate an infinite Web of documents. The motivations behind such traps vary. Anti-spam traps are designed to catch crawlers used by spammers looking for email addresses, while other sites use traps to catch search engine crawlers to boost their search ratings.
