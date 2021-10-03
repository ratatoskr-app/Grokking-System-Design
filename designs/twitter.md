# Twitter

## Summary
![overview](../img/twitter-overview.png)
![summary](../img/twitter-detail.png)

## Requirements and Goals

### Functional Requirements
1. Users should be able to post new tweets.
2. A user should be able to follow other users.
3. Users should be able to mark tweets as favorites.
4. The service should be able to create and display a user’s timeline consisting of top tweets from all the people the user follows.
5. Tweets can contain photos and videos.

### Non-functional Requirements
1. Our service needs to be highly available.
2. Acceptable latency of the system is 200ms for timeline generation.
3. Consistency can take a hit (in the interest of availability); if a user doesn’t see a tweet for a while, it should be fine.

### Extended Requirements
1. Searching for tweets.
2. Replying to a tweet.
3. Trending topics – current hot topics/searches.
4. Tagging other users.
5. Tweet Notification.
6. Who to follow? Suggestions?
7. Moments. 

### 3. Capacity Estimation and Constraints
Let’s assume we have one billion total users with **200 million** daily active users (DAU). Also assume we have **100 million new tweets** every **day** and on average each **user follows 200 people.**

**How many favorites per day?** 
If, on average, each user favorites five tweets per day we will have:
`200M users * 5 favorites => 1B favorites`

**How many total tweet-views will our system generate?** 
Let’s assume on average a user visits their timeline two times a day and visits five other people’s pages. On each page if a user sees 20 tweets,
then our system will generate 28B/day total tweet-views:

`200M DAU * ((2 + 5) * 20 tweets) => 28B/day`

**Storage Estimates** Let’s say each tweet has 140 characters and we need two bytes to store a character without compression. Let’s assume we need 30 bytes to store metadata with each tweet (like ID, timestamp, user ID, etc.). Total storage we would need:

`100M * (280 + 30) bytes => 30GB/day`

What would our storage needs be for five years? How much storage we would need for users’ data, follows, favorites? We will leave this for the exercise.
Not all tweets will have media, let’s assume that on average every fifth tweet has a photo and every tenth has a video. Let’s also assume on average a photo is 200KB and a video is 2MB. This will lead us to have 24TB of new media every day.

`(100M/5 photos * 200KB) + (100M/10 videos * 2MB) ~= 24TB/day`

**Bandwidth Estimates** Since total ingress is 24TB per day, this would translate into 290MB/sec.
Remember that we have 28B tweet views per day. We must show the photo of every tweet (if it has a photo), but let’s assume that the users watch every 3rd video they see in their timeline. So, total egress will be:

```
(28B * 280 bytes) / 86400s of text => 93MB/s
(28B/5 * 200KB ) / 86400s of photos => 13GB/S
(28B/10/3 * 2MB ) / 86400s of Videos => 22GB/s
Total ~= 35GB/s
```

## System APIs
- Posting new Tweet
`tweet(api_dev_key, tweet_data, tweet_location, user_location, media_ids, maximum_results_to_return)`

**Parameters:**
`api_dev_key (string)`: The API developer key of a registered account. This will be used to, among other things, throttle users based on their allocated quota.
`tweet_data (string):` The text of the tweet, typically up to 140 characters.
`tweet_location (string)`: Optional location (longitude, latitude) this Tweet refers to. 
`user_location(string)`: Optional location (longitude, latitude) of the user adding the tweet.
`media_ids (number[])`: Optional list of media_ids to be associated with the Tweet. (All the media photo, video, etc. need to be uploaded separately).

**Returns: (string)**
A successful post will return the URL to access that tweet. Otherwise, an appropriate HTTP error is
returned.

## High Level System Design

- Write Rate : 1160/s
- Read Rate : 325K/s 

![Screen Shot 2021-10-03 at 4 56 30 PM](https://user-images.githubusercontent.com/1195878/135755633-ee371a8a-da53-48fe-89a5-bcf5ec0f223f.png)

## Data Sharding

- **Sharding based on UserID:**
  - Hot Users ?
  - Over times some users have lots of data (non-uniform distribution)
- **Sharding based on TweetID:**  Each tweet will be on one server (we need some aggregator service)
  - Our application (app) server will find all the people the user follows.
  - App server will send the query to all database servers to find tweets from these people.
  - Each database server will find the tweets for each user, sort them by recency and return the top tweets.
  - App server will merge all the results and sort them again to return the top results to the user. 
- **Sharding based on Tweet creation time:** Storing tweets based on creation time will give us the advantage of fetching all the top tweets quickly and we only have to query a very small set of servers. The problem here is that the traffic load will not be distributed, e.g., while writing, all new tweets will be going to one server and the remaining servers will be sitting idle. Similarly, while reading, the server holding the latest data will have a very high load as compared to servers holding old data.
- **What if we can combine sharding by TweedID and Tweet creation time?**
  - In the above approach, we still have to query all the servers for timeline generation, but our reads (and writes) will be substantially quicker.
  - Since we don’t have any secondary index (on creation time) this will reduce our write latency
  - While reading, we don’t need to filter on creation-time as our primary key has epoch timeincluded in it. 
![Screen Shot 2021-10-03 at 5 06 03 PM](https://user-images.githubusercontent.com/1195878/135755960-2b343d22-1e37-4373-8580-b57dba789001.png)

## Cache 
- When the cache is full and we want to replace a tweet with a newer/hotter tweet, how would we choose? : LRU
- How can we have a more intelligent cache? If we go with 80-20 rule, that is 20% of tweets generating 80% of read traffic which means that certain tweets are so popular that a majority of people read them. This dictates that we can try to cache 20% of daily read volume from each shard.
- What if we cache the latest data?

![Screen Shot 2021-10-03 at 5 09 49 PM](https://user-images.githubusercontent.com/1195878/135756085-d1db4702-4435-4ee4-b30d-b13e32675442.png)

## Timeline Generation : ( Pre-Computed )

## Replication and Fault Tolerance 
Since our system is read-heavy, we can have multiple secondary database servers for each DB partition.Secondary servers will be used for read traffic only. All writes will first go to the primary server and then will be replicated to secondary servers. This scheme will also give us fault tolerance, since whenever the primary server goes down we can failover to a secondary server.

##  Monitoring
Having the ability to monitor our systems is crucial. We should constantly collect data to get an instant insight into how our system is doing. We can collect following metrics/counters to get an understanding of the performance of our service:
1. New tweets per day/second, what is the daily peak?
2. Timeline delivery stats, how many tweets per day/second our service is delivering.
3. Average latency that is seen by the user to refresh timeline.

By monitoring these counters, we will realize if we need more replication, load balancing, or caching.

## Extended Requirements
### How do we serve feeds? 
Get all the latest tweets from the people someone follows and merge/sort them by time. Use pagination to fetch/show tweets. Only fetch top N tweets from all the people someone follows. This N will depend on the client’s Viewport, since on a mobile we show fewer tweets compared to a Web client. We can also cache next top tweets to speed things up. Alternately, we can pre-generate the feed to improve efficiency; for details please see ‘Ranking and timeline generation’ under Designing Instagram.

### Retweet: 
With each Tweet object in the database, we can store the ID of the original Tweet and not store any contents on this retweet object.

### Trending Topics: 
We can cache most frequently occurring hashtags or search queries in the last N seconds and keep updating them after every M seconds. We can rank trending topics based on the frequency of tweets or search queries or retweets or likes. We can give more weight to topics which are
shown to more people.

### Who to follow? How to give suggestions? 
This feature will improve user engagement. We can suggest friends of people someone follows. We can go two or three levels down to find famous people for the
suggestions. We can give preference to people with more followers. As only a few suggestions can be made at any time, use Machine Learning (ML) to shuffle and reprioritize. ML signals could include people with recently increased follow-ship, common followers if the other person is following this user, common location or interests, etc.

### Moments: 
Get top news for different websites for past 1 or 2 hours, figure out related tweets, prioritize them, categorize them (news, support, financial, entertainment, etc.) using ML – supervised learning or Clustering. Then we can show these articles as trending topics in Moments.





