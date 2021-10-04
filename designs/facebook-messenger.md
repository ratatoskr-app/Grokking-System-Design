# Facebook Messenger

## 1.Summary
![overview](../img/facebook-messenger-overview.png)
![summary](../img/facebook-messenger-detail.png)

## 2.Requirements and Goals

- **Functional Requirements**
  - Messenger should support one-on-one conversations between users.
  - Messenger should keep track of the online/offline statuses of its users.
  - Messenger should support persistent storage of chat history

- **Non-functional Requirements**
  - Users should have real-time chat experience with minimum latency.
  - Our system should be highly consistent; users should be able to see the same chat history on all their devices.
  - Messenger’s high availability is desirable; we can tolerate lower availability in the interest of consistency. 

- **Extended Requirements**
  - **Group Chats:** Messenger should support multiple people talking to each other in a group.
  - **Push notifications:** Messenger should be able to notify users of new messages when they are offline. 

## 3.Capacity Estimation and Constraints
- Assumptions
  - **500 million** daily active users
  - 40 Message/Day each user
  - `500M * 40 => 20B/Day`
- Storage Estimation
  - Average a message is 100 bytes
  - 2TB of storage per day `20 billion messages * 100 bytes => 2 TB/day`
  - 3.6 PB for 5 years `2 TB * 365 days * 5 years ~= 3.6 PB`
  - Space for Users Infromation and message metadata
- Bandwidth Estimation:
  - 25MB/S `2 TB / 86400 sec ~= 25 MB/s` both upload and download

**High level estimates:**

- Total messages 20 billion per day
- Storage for each day 2TB
- Storage for 5 years 3.6PB
- Incomming data 25MB/s
- Outgoing data 25MB/s

## 4. High Level Design
At a high-level, we will need a chat server that will be the central piece, orchestrating all the communications between users. When a user wants to send a message to another user, they will connect to the chat server and send the message to the server; the server then passes that message to the other user and also stores it in the database.

![Screen Shot 2021-10-03 at 3 54 19 PM](https://user-images.githubusercontent.com/1195878/135753527-22905bc9-3f14-4d42-a492-5478c752ffdd.png)

## 5. Detailed Component Desgin

### Simple Solution
Our system at the high level needs to handle the following use cases :

1. Receive incoming messages and deliver outgoing messages.
2. Store and retrieve messages from the database.
3. Keep a record of which user is online or has gone offline, and notify all the relevant users about these status changes. 

### a. Message Handling
**How would we efficiently send/receive messages?**

- To Send message, a user need to connect to the server and post messages for the other users
- To get a message from server, the user has two options
  -  Pull Model : User can peridically ask the server if there are any new message for them (polling)
  -  Push Model : user can keep a connection open with server (long polling, websockets)
 
#### Qustion / Answers 
- **How can the server keep track of all the opened connection to redirect messages to the users efficiently?**
- The server can maintain a hash table, where “key” would be the UserID and “value” would be the connection object. So whenever the server receives a message for a user, it looks up that user in the hash table to find the connection object and sends the message on the open request.

- **What will happen when the server receives a message for a user who has gone offline?**
- If the receiver has disconnected, the server can notify the sender about the delivery failure. If it is atemporary disconnect, e.g., the receiver’s long-poll request just timed out, then we should expect a reconnect from the user. In that case, we can ask the sender to retry sending the message. This retry could be embedded in the client’s logic so that users don’t have to retype the message. The server can also store the message for a while and retry sending it once the receiver reconnects.

- **How many chat servers we need?** 
- Let’s plan for 500 million connections at any time. Assuming a modern server can handle 50K concurrent connections at any time, we would need 10K such servers.

- **How do we know which server holds the connection to which user?**
- We can introduce a software load balancer in front of our chat servers; that can map each UserID to a server to redirect the request.

- **How should the server process a ‘deliver message’ request?**
- The server needs to do the following things upon receiving a new message: 
  - 1) Store the message in the database 
  - 2) Send the message to the receiver and 
  - 3) Send an acknowledgment to the sender.

The chat server will first find the server that holds the connection for the receiver and pass the message
to that server to send it to the receiver. The chat server can then send the acknowledgment to the
sender; we don’t need to wait for storing the message in the database (this can happen in the
background). Storing the message is discussed in the next section.

### b.Storing and retrieving the messages from the database

**Store Message to Database Procedure:**

1. **Option 1:** Start a separate thread, which will work with the database to store the message.
2. **Option 2:** Send an asynchronous request to the database to store the message. 

**Which storage system to use?**

- **We cannot use RDBMS like MySQL or NoSQL like MongoDB** because we cannot afford to read/write a row from the database every time a user receives/sends a message.
- This will not only make the basic operations of our service run with high latency, but also create a huge load on databases.
- We should use wide-colun database solution like **HBase** or **Casandra**

**Why HBase?**
HBase is modeled after Google’s BigTable and runs on top of Hadoop Distributed File System (HDFS). 
- HBase groups data together to store new data in a memory buffer and, once the buffer is full, it dumps the data to the disk. This way of storage not only helps storing a lot of small data  quickly, but also fetching rows by the key or scanning ranges of rows. 
- HBase is also an efficient database to store variably sized data, which is also required by our service

### c. Managing User's status
We need to keep track of user’s online/offline status and notify all the relevant users whenever a status change happens. 

Since we are maintaining a connection object on the server for all active users, we can easily figure out the user’s current status from this. 

With 500M active users at any time, if we have to  broadcast each status change to all the relevant active users, it will consume a lot of resources. We can do the following optimization around this:

1. Whenever a client starts the app, it can pull the current status of all users in their friends’ list.
2. Whenever a user sends a message to another user that has gone offline, we can send a failure to the sender and update the status on the client.
3. Whenever a user comes online, the server can always broadcast that status with a delay of a few seconds to see if the user does not go offline immediately.
4. Client’s can pull the status from the server about those users that are being shown on the user’s viewport. This should not be a frequent operation, as the server is broadcasting the online status of users and we can live with the stale offline status of users for a while.
5. Whenever the client starts a new chat with another user, we can pull the status at that time. 
6. We can also implement a Heartbeat progress to make sure we dont overload the server with requests

![Screen Shot 2021-10-03 at 4 35 13 PM](https://user-images.githubusercontent.com/1195878/135754896-9eb51f23-8189-4362-b9dd-a64502a75c86.png)

**Design Summary:** Clients will open a connection to the chat server to send a message; the server will then pass it to the requested user. All the active users will keep a connection open with the server to receive messages. Whenever a new message arrives, the chat server will push it to the receiving user on the long poll request. Messages can be stored in HBase, which supports quick small updates, and range based searches. The servers can broadcast the online status of a user to other relevant users. Clients can pull status updates for users who are visible in the client’s viewport on a less frequent basis.

# 6.Data partitioning
Since we will be storing a lot of data (3.6PB for five years), we need to distribute it onto multiple database servers. 

**What will be our partitioning scheme?**

**Partitioning based on UserID:** 
- Use UserID as sharding key
- Assume 4TB as size of DB shard
- 3.6PB/4TB -> 1000 shards
- `UserID % 1000`
- Chat history will be on a single shard -> quick fetch
- We can start with smaller partitions

**Partitioning based on MessageID:** (Not a good option) 
- If we store different messages of a user on separate database shards, fetching a range of messages of a chat would be very slow, so we should not adopt this scheme.

## 7. Cache
We can cache a few recent messages (say last 15) in a few recent conversations that are visible in a user’s viewport (say last 5). Since we decided to store all of the user’s messages on one shard, cache for a user should entirely reside on one machine too.

## 8. Load balancing
We will need a load balancer in front of our chat servers; that can map each UserID to a server that holds the connection for the user and then direct the request to that server. Similarly, we would need a load balancer for our cache servers.

## 9. Fault tolerance and Replication
**What will happen when a chat server fails?**

Our chat servers are holding connections with the users.
If a server goes down, should we devise a mechanism to transfer those connections to some other server? It’s extremely hard to failover TCP connections to other servers; an easier approach can be to have clients automatically reconnect if the connection is lost.

**Should we store multiple copies of user messages?** 
We cannot have only one copy of the user’s data, because if the server holding the data crashes or is down permanently, we don’t have any mechanism to
recover that data. For this, either we have to store multiple copies of the data on different servers or use techniques like Reed-Solomon encoding to distribute and replicate it.

## 10. Extended Requirements

### a. Group chat
We can have separate group-chat objects in our system that can be stored on the chat servers. A groupchat object is identified by GroupChatID and will also maintain a list of people who are part of that chat. Our load balancer can direct each group chat message based on GroupChatID and the server handling that group chat can iterate through all the users of the chat to find the server handling the connection of each user to deliver the message.

In databases, we can store all the group chats in a separate table partitioned based on GroupChatID.




