# Instagram

## Summary
![overview](../img/instagram-overview.png)
![summary](../img/instagram-detail.png)

## Requirements

- **Functional Requirements**
  - Users should be able to upload/download/view photos.
  - Users can perform searches based on photo/video titles.
  - Users can follow other users.
  - The system should be able to generate and display a user’s News Feed consisting of top photos
from all the people the user follows.

- **Non-functional Requirements**
  - Our service needs to be highly available.
  - The acceptable latency of the system is 200ms for News Feed generation.
  - Consistency can take a hit (in the interest of availability), if a user doesn’t see a photo for a
while; it should be fine.
  - The system should be highly reliable; any uploaded photo or video should never be lost

## Some Design Considerations
- Read-Heavy System - Get Photos quickly
- Efficient management of storage (users can upload many photos)
- Low latency is expected while viewing photos.
- Data should be 100% reliable. no photo should be lost

## Capacity Estimation and Constraints
- Let’s assume we have 500M total users, with 1M daily active users.
- 2M new photos every day => `2M/(24*3600) => 23 new photos every second`
- Average photo file size => 200KB
- Total space required for 1 day of photos
  - `2M * 200KB => 400 GB`
- Total space required for 10 years
  - `400GB * 365 (days a year) * 10 (years) ~= 1425TB`

## High Level Design
- Scenarios:
  - Upload photos 
  - View and search photos. 
- Need some object storage servers to store photos 
- Database to store metadata information about the photos.

![High Level](https://user-images.githubusercontent.com/1195878/135751947-cef7afe6-b893-41a2-9180-2679d240f31e.png)

## Database Schema
- We need to store data about 
  - Users
  - Photos
  - Follows

![Database](https://user-images.githubusercontent.com/1195878/135751930-d38d5ba6-10d9-46d8-8f16-acc34105be99.jpg)

- RDBMS and good for joins but are challenging ( hard to scale )
- We can store photos in distribured file storage like HDFS or S3
- We can store the above schema in a distributed key-value store to enjoy the benefits offered by NoSQL.
- We can user wide-column datastore like HBase and Cassandra 
  - Key: `UserID` , Value : `List of PhotoIDs of users`
  - Key: `UserID` , Value : `List of users that he/she follows` 
  - Always maintain a certain number of replicas to offer reliability
  - Deletes don’t get applied instantly, data is retained for certain days 


## Data Size Estimation

- User (68 bytes)
  - Fields
    - UserID (4 bytes)
    - Name (20 bytes)
    - Email (32 bytes)
    - DateOfBirth (4 bytes)
    - CreationDate (4bytes)
    - LastLogin (4 bytes)
  - Total : `500M * 68 = 32GB`
- Photos
  - Fields (284 bytes)
    - PhotoID (4 bytes)
    - UserID (4 bytes)
    - PhotoPath (256 bytes)
    - PhotoLatitude (4 bytes)
    - PhotLongitude(4 bytes)
    - UserLatitude (4 bytes)
    - UserLongitude (4 bytes)
    - CreationDate (4 bytes)
  - Total : `2M * 284 bytes ~= 0.5GB per day`
- Follows
  -  Assume each user follows 500 users
  -  `500 million users * 500 followers * 8 bytes ~= 1.82TB`

**Total space required for all tables for 10 years will be 3.7TB:**

`32GB + 1.88TB + 1.82TB ~= 3.7TB`

## Component Design

As Photo upload ( write ) could be slow and read should be fast we should consider separating these into separate services


## Reliability and Redundancy
- Losing files is not an option, Multiple copies of files ( storage layer )
- Multiple replicas of services ( High availability, Redundancy remove SPOF)

![Reliability](https://user-images.githubusercontent.com/1195878/135752007-eee91c1a-c183-4ff7-aad6-af562123661e.png)


## Data Sharding

Let’s discuss different schemes for metadata sharding:

### a. Partitioning based on UserID ###
- Assume DB shard is 1TB, 3.7TB will be 4 shards
- Assume 10 shard for better performance and scalability
- UserID % 10 -> Find shard number
- Add shard number to each PhotoID to make it unique throughout system `1-1234, 2-1234, 3-1234`
- Issues:
  - How would we handle hot users? 
  - Users with lot of photos => a non-uniform distribution of storage.
  - User photos does not fit in 1 shard => we distribute photos of a user onto multiple shards => higher latencies?
  - Storing all photos of a user on one shard
    - Unavailability of all of the user’s data if that shard is down 
    - Higher latency if it is serving high load

### b. Partitioning based on PhotoID ### 
- PhotoID % 10 -> Shard Key
- No need to append ShardId and PhotoID
- We dont have `PhotoId` to calculate the shard ? 
  - We can use a separete database to generate auto-incr ids => SPOF
  - We can use custom rules to `auto-increment-increment` and `auto-increment-offset`


**How can we plan for the future growth of our system?** 
We can have a large number of logical partitions to accommodate future data growth, such that in the beginning, multiple logical partitions
reside on a single physical database server. Since each database server can have multiple database
instances on it, we can have separate databases for each logical partition on any server. So whenever
we feel that a particular database server has a lot of data, we can migrate some logical partitions from it
to another server. We can maintain a config file (or a separate database) that can map our logical
partitions to database servers; this will enable us to move partitions around easily. Whenever we want
to move a partition, we only have to update the config file to announce the change.

## Ranking and News Feed Generation

1. Get Photos based on the follow list
2. Rank them based on recency, likeness, etc

### Solutions:

**Pre-generating the News Feed:** 
1. We can have dedicated servers that are continuously generating users’ News Feeds and storing them in a ‘UserNewsFeed’ table. 
2. So whenever any user needs the latest photosfor their News Feed, we will simply query this table and return the results to the user.
3. Whenever these servers need to generate the News Feed of a user, they will first query the UserNewsFeed table to find the last time the News Feed was generated for that user. 
4. New News Feed data **(afterward that time)** will be generated from that time onwards (following the steps mentioned above).

### What are the different approaches for sending News Feed contents to the users? ###

**1. Pull:** Clients can pull the News Feed contents from the server on a regular basis or manually

- Possible Issues 
  - a) New data might not be shown to the users until clients issue a pull request 
  - b) Most of the time pull requests will result in an empty response if there is no new data.

**2. Push**: Servers can push new data to the users as soon as it is available. `Long Poll request`
- Possible Issues
  - a user who follows a lot of people
  - celebrity user who has millions of followers
  - In this case, the server has to push updates quite frequently.

**3. Hybrid:** We can adopt a hybrid approach. 
- We can move all the users who have a high number of follows to a pull-based model
- Push model to those users who have a few hundred (or thousand) follows. 
- Another approach could be that the server pushes updates to all the users not more than acertain frequency, letting users with a lot of follows/updates to regularly pull data.

## News Feed Creating With Sharded Data 

**How to sort photos based on creation date?**
- We can use epoch time for this. Let’s say our PhotoID will have two parts
  - First part will be representing epoch time
  - Second part will be an auto-incrementing sequence from our keygenerating DB

**What could be the size of our PhotoID?** 
Let’s say our epoch time starts today, how many bits we would need to store the number of seconds for next 50 years?
`86400 sec/day * 365 (days a year) * 50 (years) => 1.6 billion seconds`

***We would need 31 bits to store this number.***
Since on the average, we are expecting 23 new photos per second; 
we can allocate 9 bits to store auto incremented sequence. So every second we can store (2^9 => 512) new photos. 
We can reset our auto incrementing sequence every second.

## Cache and Load balancing
- CDN for photo cache server (geographically)
- Cache for metadata servers to cache hot database rows (LRU for cache eviction)

**How can we build more intelligent cache?** 
If we go with 80-20 rule, i.e., 20% of daily read volume for photos is generating 80% of traffic which means that certain photos are so popular that the majority of people read them. This dictates that we can try caching 20% of daily read volume of photos and metadata

