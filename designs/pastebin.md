# Pastebin

## 1. Summary
![overview](../img/pastebin-overview.png)
![summary](../img/pastebin-detail.png)

- Database
  - Metadata Database : RDBMS
  - Object Storage : S3
  - Store Object Storage key in database
- Slug Generation
  - Random string with collision detection
  - KGS ( Two tables )

## 2. Requirements and Goals of the System

Our Pastebin service should meet the following requirements:

- **Functional Requirements:**
  - Users should be able to upload or “paste” their data and get a unique URL to access it.
  - Users will only be able to upload text.
  - Data and links will expire after a specific timespan automatically; users should also be able to
  specify expiration time.
  - Users should optionally be able to pick a custom alias for their paste.

- **Non-Functional Requirements:**
  - The system should be highly reliable, any data uploaded should not be lost.
  - The system should be highly available. This is required because if our service is down, users
  will not be able to access their Pastes.
  - Users should be able to access their Pastes in real-time with minimum latency.
  - Paste links should not be guessable (not predictable).

- **Extended Requirements:**
  - Analytics, e.g., how many times a paste was accessed?
  - Our service should also be accessible through REST APIs by other services.

## 3. Some Design Considerations
Pastebin shares some requirements with URL Shortening service, but there are some additional design considerations we should keep in mind.

**What should be the limit on the amount of text user can paste at a time?** 

We can limit users not to have Pastes bigger than 10MB to stop the abuse of the service.

**Should we impose size limits on custom URLs?** 

Since our service supports custom URLs, users can pick any URL that they like, but providing a custom URL is not mandatory. However, it is reasonable (and often desirable) to impose a size limit on custom URLs, so that we have a consistent URL database.


## 4. Capacity Estimation and Constraints

- Assumption 
  - Read Heady : there will be more read requests compared to new Pastes creation.
  - Assume **5:1** ratio between read and write.

- Traffic estimates
  - **1M** new pastes per day or **30M** new per month
  - New Pasters per seconds
    - `1M / (24 hours * 3600 seconds) ~= 12 pastes/sec`
  - **5M** Paste reads per day or **150M** read per month
    - `5M / (24 hours * 3600 seconds) ~= 58 reads/sec`

- Storage estimates
  - Users can upload maximum **10MB** of data but let's assume on average **10KB**
  - Total Objects : `1M * 365 * 10 = 3.6B`
  - Object Storage O over **10 years** `1M * 10KB => 10 GB/day` for 10 years it will be `36TB`
  - Unique Keys : if we base64 encode a-Z, A-Z, 0-9,.,- with **6 letters** we have `64^6 ~= 68.7 billion unique strings`
  - Keys Storage : We need a byte per char, with **6 letters** will be `3.6B * 6 => 22 GB`
  - Total Storage : `(22GB + 36TB) / 0.7 = 51.4TB` 70% capacity model

- Bandwidth estimates
  - Write Requests: `12 * 10KB => 120 KB/s`
  - Read Requests: `58 * 10KB => 0.6 MB/s`

- Memory estimates:
  - Followw the 80-20 rule assuming 20% of pastes generate 80% of traffic cache 20% of hot pastes `0.2 * 5M * 10KB ~= 10 GB`

## 5. System APIs
We can have SOAP or REST APIs to expose the functionality of our service. Following could be the definitions of the APIs to **create**/**retrieve**/**delete** Pastes:

### `addPaste`

- Parameters

| Name 	| Type 	| Note 	|
|---	|---	|---	|
| api_dev_key 	| (string) 	| The API developer key of a registered account. This will be used to, among other things, throttle users based on their allocated quota. 	|
| paste_data 	| (string) 	| Textual data of the paste. 	|
| custom_url 	| (string) 	| Optional custom URL. 	|
| user_name 	| (string) 	| Optional user name to be used to generate URL. paste_name (string): Optional name of the paste 	|
| expire_date 	| (string) 	| Optional expiration date for the paste. 	|

- Returns: 
  - (string)
  - A successful insertion returns the URL through which the paste can be accessed, otherwise, it will return an error code. Similarly, we can have retrieve and delete Paste APIs:

### `getPaste(api_dev_key, api_paste_key)`

Where “api_paste_key” is a string representing the Paste Key of the paste to be retrieved. This API will return the textual data of the paste.

### `deletePaste(api_dev_key, api_paste_key)`

A successful deletion returns ‘true’, otherwise returns ‘false’.

## 6. Database Design

- Observation
  - We need to store billions of records.
  - Each metadata object we are storing would be small (less than 100 bytes).
  - Each paste object we are storing can be of medium size (it can be a few MB).
  - There are no relationships between records, except if we want to store which user created what Paste.
  - Our service is read-heavy.
- Schema 

#### Paste

| Column          |   Type       |           
|-----------------|--------------|
| URLHash         | varchar(16)  |
| ContentKey      | varchar(255) |
| ExpireationDate | datetime     |
| CreationDate    | datetime     |

#### User

| Column       |    Type     |
|--------------|-------------|
| UserId       | int         |
| Name         | varchar(20) |
| Email        | varchar(32) |
| CreationDate | datetime    |
| LastLogin    | datetime    |

Here, ‘URlHash’ is the URL equivalent of the TinyURL and ‘ContentKey’ is the object key storing the contents of the paste.


## 7. High Level Design
At a high level, we need an application layer that will serve all the read and write requests. Application layer will talk to a storage layer to store and retrieve data. We can segregate our storage layer with one database storing metadata related to each paste, users, etc., while the other storing the paste contents in some object storage (like Amazon S3). This division of data will also allow us to scale them individually.

![Screen Shot 2021-10-04 at 11 13 29 AM](https://user-images.githubusercontent.com/1195878/135812403-69b853e0-46dc-467b-b30a-56617062f162.png)

## 8. Component Design

### a. Application layer

Our application layer will process all incoming and outgoing requests. The application servers will be talking to the backend data store components to serve the requests.

#### Write Request
- Random Key Generation
  - Generating a six-letter random string if not a custom key is provided
    - Problems
      - Duplicate key, keeps generating till we have a unique key
      - Return an error if custom key is already in database
  - Key Generation Service
    - Generate random 6 letter strings and store them in database (keys DB)
    - When a keys is needed, take one from the keys DB
    - Two tables to store keys, one for used and one for not used
    - Problems
      - Single point of failure, Solution : standby replica of KGS
      - App Server can cache some keys from keys-DB, in case it dies we are losing those keys

#### Read Request
- Upon receiving a read paste request, the application service layer contacts the datastore
- The datastore searches for the key, and if it is found, returns the paste’s contents. Otherwise, an error code is returned.

### b. Datastore layer
We can divide our datastore layer into two:

1. **Metadata database:** We can use a relational database like MySQL or a Distributed Key-Value store like Dynamo or Cassandra.
2. **Object storage:**  Object Storage like Amazon’s S3. Whenever we feel like hitting our full capacity on content storage, we can easily increase it by adding more servers.

![Screen Shot 2021-10-04 at 11 22 58 AM](https://user-images.githubusercontent.com/1195878/135813736-c10ea2a8-167d-4da2-9340-886b40ebc001.png)


## 9. Purging or DB Cleanup
Please see Designing a [URL Shortening service](https://github.com/ratatoskr-app/Grokking-System-Design/blob/master/designs/short-url.md). 

## 10. Data Partitioning and Replication
Please see Designing a [URL Shortening service](https://github.com/ratatoskr-app/Grokking-System-Design/blob/master/designs/short-url.md).

## 11. Cache and Load Balancer
Please see Designing a [URL Shortening service](https://github.com/ratatoskr-app/Grokking-System-Design/blob/master/designs/short-url.md).

## 12. Security and Permissions
Please see Designing a [URL Shortening service](https://github.com/ratatoskr-app/Grokking-System-Design/blob/master/designs/short-url.md).






