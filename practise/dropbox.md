# Design Dropbox

Dropbox is a cloud-based file storage service that allows users to store and share files. It
provides a secure and reliable way to store and access files from anywhere, on any device.


# Solution

## Assumptions
(We use the assumptions to scope out what we intend to build in the system)
1. User can upload file and view it from multiple devices 
2. Files can b e shared with other users
3. Offline download is for later -- Will not explore if the local folder is in sync with cloud folder/ File has changed etc for now (revisit later)
4. Directory / File organization is for later -- for now the UI is just a list of files shown to the user
5. Sharing permissions are for each file 
6. Geography constraints ignored for now
7. No versioning? No merge if two uploads land  -- latest document wins


## Approximations
(We use the approximations to scope out the scale of the system)

10M DAU, uploading 3-5 files daily, each file is max 1GB. Users access ~30-50 files daily (read ~ 10X write basically)

Daily disk size required: 10 * 1e6 * 5 * 1e9 = 5 *1e16 ~ 0.05 PB / day (18 PB / yer)

At the lower end, if we assume a 100 MB file, it is 1.8 PB/ year, so maybe ~7 EB / year?
 
Write QPS: 10 * 1e6 * 5 / 1e5 = 500 writes / second 
Read QPS: 5000 reads/ second  (I dont remember the rule of thumb but maybethe conclusion is that we don't need caches/ sharding etc)




## Constraint
(We use the constraints to guid how to design the system) 
1. Use a industry standard blob storage system for file storage -- will rely on their backup strategy, but the companies explicit backup strategy could be multi-cloud blob storage (budget is the main constraint here )

2. Within a working session, we want the files to be availalbe with low latency (e.g., user is working on two devices or there is a collaborative download-modify-upload) [Not sure about this ]

3. We want consistency -- each user should get the "latest" file tht is stored. 

## Incentives (Trade-Offs)
(We use the trade-off sections to further refine the design)
I think the tension is between  2 and 3 above. In the sense that consistency is achieved when the table is UPDATED after the changes are committed to Blob. In any case, we might have to think about our metata adDB size (not the Blob storage size) -- if we can fit it in one DB (with backup) and we do not need distributed DB storage for the metadata, o then we should be able to achieve both 2 and 3 (with the only unvoidable latency being the time it takes to save the blob)

500 writes / sec * (USER Id, File Id, Blob Id), and separtely File ID, Allowed UserID table. If the IDs are all Int64, then how much DB space do we need? (need to memorize more math here) -- Assume that we can get it in 1 Shard.


## Decisons / Design
### APIs
1. Upload (uploads a file)
2. Download
3. Delete
4. Share

### DBs 
File Metadata :  File Id, File Name, File Type, Blo b URL (index on File ID)
User MetData: User Id, File_id, is_owner  (All files that the user owns/ is shared with)
(Index: User ID)
User : User Id --> Email Address

### High level

Upload Service -- Gets User Id, File --> Writes to blob --> Gets back Blob URL --> updates File and User table

Download Service -- Gets User Id --> Reads User Table ---> Displays link to all Files --> download happens from Blob

Delete -- Same as Download service --- maybe temporarily mark file as deleted on table (and a background daemon actually deletes the rows and blob later -- this way, we can also allow some sort of "restore" function -- e.g. daemon only deletes it if file is deleted for 10 days)

Share -- Same as Download (display files) --User can add email addreess -- create a new user id if email address is not there --- add to User MetData for the file ID -- retrieve email address and send email.

## Failure Mode
(Deep dive on one or two failure modes)

## Metrics and Monitoring
(Discussion on what metrics to collect and how to use the system to debug)