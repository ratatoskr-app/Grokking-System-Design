# Youtube

## 1. Summary
![overview](../img/youtube-overview.png)
![detail](../img/youtube-detail.png)


## 2. Requirements and Goals of the System
For the sake of this exercise, we plan to design a simpler version of Youtube with following requirements:
### Functional Requirements:
1. Users should be able to upload videos.
2. Users should be able to share and view videos.
3. Users should be able to perform searches based on video titles.
4. Our services should be able to record stats of videos, e.g., likes/dislikes, total number of views, etc.
5. Users should be able to add and view comments on videos.

### Non-Functional Requirements:
1. The system should be highly reliable, any video uploaded should not be lost.
2. The system should be highly available. Consistency can take a hit (in the interest of availability); if a user doesn’t see a video for a while, it should be fine.
3. Users should have a real time experience while watching videos and should not feel any lag.

### Not in scope: 
Video recommendations, most popular videos, channels, subscriptions, watch later, favorites, etc.

## 3. Capacity Estimation and Constraints
Let’s assume we have 1.5 billion total users, 800 million of whom are daily active users. If, on average, a user views five videos per day then the total video-views per second would be:

`800M * 5 / 86400 sec => 46K videos/sec`

Let’s assume our upload:view ratio is 1:200, i.e., for every video upload we have 200 videos viewed, giving us 230 videos uploaded per second.

`46K / 200 => 230 videos/sec`

**Storage Estimates:** 
Let’s assume that every minute 500 hours worth of videos are uploaded to Youtube. If on average, one minute of video needs 50MB of storage (videos need to be stored in multiple formats), the total storage needed for videos uploaded in a minute would be:

`500 hours * 60 min * 50MB => 1500 GB/min (25 GB/sec)`

These numbers are estimated with ignoring video compression and replication, which would change our estimates.

**Bandwidth estimates:** With 500 hours of video uploads per minute and assuming each video upload takes a bandwidth of 10MB/min, we would be getting 300GB of uploads every minute.

`500 hours * 60 mins * 10MB => 300GB/min (5GB/sec)`

Assuming an upload: view ratio of 1:200, we would need 1TB/s outgoing bandwidth.

## 4. System APIs
We can have SOAP or REST APIs to expose the functionality of our service. The following could be the definitions of the APIs for uploading and searching videos:

### Upload Video

`uploadVideo(api_dev_key, video_title, vide_description, tags[], category_id, default_language, recording_details, video_contents)`

**Parameters:**
api_dev_key (string): The API developer key of a registered account. This will be used to, among other things, throttle users based on their allocated quota.
video_title (string): Title of the video.
vide_description (string): Optional description of the video.
tags (string[]): Optional tags for the video.
category_id (string): Category of the video, e.g., Film, Song, People, etc.
default_language (string): For example English, Mandarin, Hindi, etc.
recording_details (string): Location where the video was recorded.
video_contents (stream): Video to be uploaded.

**Returns: (string)**
A successful upload will return HTTP 202 (request accepted) and once the video encoding is completed the user is notified through email with a link to access the video. We can also expose a queryable API to let users know the current status of their uploaded video.

### Search Video

`searchVideo(api_dev_key, search_query, user_location, maximum_videos_to_return,page_token)`

**Parameters:**
api_dev_key (string): The API developer key of a registered account of our service.
search_query (string): A string containing the search terms.
user_location (string): Optional location of the user performing the search.
maximum_videos_to_return (number): Maximum number of results returned in one request.
page_token (string): This token will specify a page in the result set that should be returned.

**Returns: (JSON)**
A JSON containing information about the list of video resources matching the search query. Each video
resource will have a video title, a thumbnail, a video creation date, and a view count.

### Stream Video

`streamVideo(api_dev_key, video_id, offset, codec, resolution)`

**Parameters:**
api_dev_key (string): The API developer key of a registered account of our service.
video_id (string): A string to identify the video.
offset (number): We should be able to stream video from any offset; this offset would be a time in
seconds from the beginning of the video. If we support playing/pausing a video from multiple devices,
we will need to store the offset on the server. This will enable the users to start watching a video on any
device from the same point where they left off.
codec (string) & resolution(string): We should send the codec and resolution info in the API from the
client to support play/pause from multiple devices. Imagine you are watching a video on your TV’s
Netflix app, paused it, and started watching it on your phone’s Netflix app. In this case, you would need
codec and resolution, as both these devices have a different resolution and use a different codec.

**Returns: (STREAM)**
A media stream (a video chunk) from the given offset.

## 5.High Level Design

1. **Processing Queue:** Each uploaded video will be pushed to a processing queue to be de-queued later for encoding, thumbnail generation, and storage.
2. **Encoder:** To encode each uploaded video into multiple formats.
3. **Thumbnails generator:** To generate a few thumbnails for each video.
4. **Video and Thumbnail storage:** To store video and thumbnail files in some distributed file storage.
5. **User Database:** To store user’s information, e.g., name, email, address, etc.
6. **Video metadata storage:** A metadata database to store all the information about videos like title, file path in the system, uploading user, total views, likes, dislikes, etc. It will also be used to store all the video comments. 

## 6.Database Schema

**Video metadata storage - MySql**
Videos metadata can be stored in a SQL database. The following information should be stored with each video:
• VideoID
• Title
• Description
• Size
• Thumbnail
• Uploader/User
• Total number of likes
• Total number of dislikes
• Total number of views

For each video comment, we need to store following information:
• CommentID
• VideoID
• UserID
• Comment
• TimeOfCreation

**User data storage - MySql**
• UserID, Name, email, address, age, registration details etc. 

## 7. Detailed Component Design

### Where would videos be stored? 
Videos can be stored in a distributed file storage system like HDFS or GlusterFS / S3

### How should we efficiently manage read traffic?
We should segregate our read traffic from write traffic. Since we will have multiple copies of each video, we can distribute our read traffic on different servers. **For metadata**, we can have master-slave configurations where writes will go to master first and then gets applied at all the slaves. Such configurations can cause some staleness in data, e.g., when a new video is added, its metadata would be inserted in the master first and before it gets applied at the slave our slaves would not be able to see it; and therefore it will be returning stale results to the user.
This staleness might be acceptable in our system as it would be very short-lived and the user would be able to see the new videos after a few milliseconds.

### Where would thumbnails be stored? 

There will be a lot more thumbnails than videos. If we assume
that every video will have five thumbnails, we need to have a very efficient storage system that can
serve a huge read traffic. There will be two consideration before deciding which storage system should
be used for thumbnails:
1. Thumbnails are small files with, say, a maximum 5KB each.
2. Read traffic for thumbnails will be huge compared to videos. Users will be watching one video at a time, but they might be looking at a page that has 20 thumbnails of other videos. 

**Google Cloud Bigtable | S3**  can be a reasonable choice here as it combines multiple files into one block to store on the
disk and is very efficient in reading a small amount of data. Both of these are the two most significant
requirements of our service. Keeping hot thumbnails in the cache will also help in improving the
latencies and, given that thumbnails files are small in size, we can easily cache a large number of such
files in memory

### Video Uploads: 
Since videos could be huge, if while uploading the connection drops we should support resuming from the same point.

### Video Encoding: 
Newly uploaded videos are stored on the server and a new task is added to the processing queue to encode the video into multiple formats. Once all the encoding will be completed the uploader will be notified and the video is made available for view/sharing.


![Screen Shot 2021-10-03 at 5 33 32 PM](https://user-images.githubusercontent.com/1195878/135757169-ad1c3194-4407-4392-a967-becf2d973ee2.png)

## Metadata Sharding
- Sharding based on UserID:
  - Popular users
  - Non-Uniform distribution
- Sharding based on VideoID:
  - Needs aggregation
  - using cache to deduct aggregation

## Video Deduplication
- Block Matching, Phase Correlation

## CDN
Our service can move popular videos to CDNs:
- CDNs replicate content in multiple places. There’s a better chance of videos being closer to the user and, with fewer hops, videos will stream from a friendlier network.
- CDN machines make heavy use of caching and can mostly serve videos out of memory.

`Less popular videos (1-20 views per day) that are not cached by CDNs can be served by our servers invarious data centers.`


