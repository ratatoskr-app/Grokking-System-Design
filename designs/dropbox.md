# Dropbox

## Summary
![overview](../img/dropbox-overview.png)
![summary](../img/dropbox-detail.png)

## Requirement and Goals of the System

- Users should be able to upload and download their files/photos from any device.
- Users should be able to share files or folders with other users.
- Our service should support automatic synchronization between devices, i.e., after updating a file
on one device, it should get synchronized on all devices.
- The system should support storing large files up to a 1 GB.
- ACID-ity is required. Atomicity, Consistency, Isolation and Durability of all file operations
should be guaranteed.
- Our system should support offline editing. Users should be able to add/delete/modify files while
offline, and as soon as they come online, all their changes should be synced to the remote
servers and other online devices.

**Extended Requirements**
- The system should support snapshotting of the data, so that users can go back to any version of
the files.

## Some Desgin Condiderations

We should expect huge read and write volumes.
- Read to write ratio is expected to be nearly the same.
- Internally, files can be stored in small parts or chunks (say 4MB); 
  - this can provide a lot of benefits i.e. all failed operations shall only be retried for smaller parts of a file. 
  - If a user fails to upload a file, then only the failing chunk will be retried.
- We can reduce the amount of data exchange by transferring updated chunks only.
- By removing duplicate chunks, we can save storage space and bandwidth usage.
- Keeping a local copy of the metadata (file name, size, etc.) with the client can save us a lot ofround trips to the server.
- For small changes, clients can intelligently upload the diffs instead of the whole chunk. 

## Capacity Estimation and Constraints
- Let’s assume that we have 500M total users, and 100M daily active users (DAU).
- Let’s assume that on average each user connects from three different devices.
- On average if a user has 200 files/photos, we will have 100 billion total files.
- Let’s assume that average file size is 100KB, this would give us ten petabytes of total storage. `100B * 100KB => 10PB`
- Let’s also assume that we will have one million active connections per minute. 

## High Level Design

- At High level we need to store files and their metadata information like File Name, File Size, Directory, etc., and who this file is shared with.
- we need some servers that can help the clients to upload/download files to Cloud Storage and some servers that can facilitate updating metadata about files and users
- We also need some mechanism to notify all clients whenever an update happens so they can synchronize their files.

![Screen Shot 2021-10-03 at 3 43 31 PM](https://user-images.githubusercontent.com/1195878/135753150-3919e6b4-a2f0-438a-83ea-04748f737cf4.png)

## Component Design

### Client


