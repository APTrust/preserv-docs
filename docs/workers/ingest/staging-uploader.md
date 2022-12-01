# The Staging Uploader

The staging uploader, ingest_staging_uploader, unpacks a tarred bag from a receiving bucket and copies all of its files into an S3 staging bucket where other ingest workers can access them. For information about what's in the staging bucket and how keys are composed, see the [staging bucket](/components/s3/#staging-bucket) documentation.

By the time the staging uploader gets to work, tag files and manifests have already copied to the staging bucket under their own names (aptrust-info.txt, manifest-sha256.txt, etc.). The [metadata gatherer](pre-fetch.md) did this early on so it and other workers could access those metadata files as needed.

All other files are copied with UUID keys which are assigned by the [metadata gatherer](pre-fetch.md) (for new files) or the [reingest manager](reingest-manager.md) (for files being reingested). Tag files bound for preservation storage, including bag-info.txt and aptrust-info.txt will also be copied to the staging bucket under their permanent UUID.

When the [preservation uploader](preservation-uploader.md) copies files from staging to preservation buckets later on, all of the files will retain their UUID in the preservation storage.

## Why Wait So Long to Go to Staging?

Unpacking a bag and copying all of its files to a staging area is an expensive in terms of memory, network I/O and time. We don't want to take this step until we're sure that the bag is valid and that the reingest manager has been able to assign proper UUIDs.

## Why Use a Staging Bucket?

After files are unpacked, at least two more workers, the [format identifier](format-identifier.md) and the [preservation uploader](preservation-uploader.md), need to access them. If we unpack files to a local disk, other workers can't get to them. Local EBS volumes are also expensive.

S3 happens to work well for this write-once, read-multiple workload. (The staging uploader writes each file once to S3. Subsequent workers read the files at least once. For now, only the format identifier and the preservation uploader read files from the staging bucket. In the future, it's possible to add more workers, such as virus scanners, format migrators, etc., to the pipeline. All of these workers can be distributed and all can have multiple instances.

!!! Note

    The staging bucket is in the same AWS region as receiving buckets and
    our ingest workers. This means copying files to staging is reasonably
    fast and will not incur inter-region transfer costs.

## Resources

The staging uploader can use large amounts of memory when copying very large files because the underlying Minio library uses memory buffers for large uploads. Large bags also require a lot of network I/O.

This worker is generally limited to two go routines for uploading files because the Docker container in which it runs has limited memory. Given Minio's heavy memory usage for large-file uploads, running too many go routines at once may lead to out-of-memory exceptions.

When the staging uploader gets overloaded, we simply add new containers to spread the load.

Like all other ingest workers, the staging uploader keeps track of which tasks it has completed in Redis. If the worker dies or requeues the task due to network errors (which are fairly common), a new worker can pick up where the old one left off without having to duplicate expensive work.

## External Services

| Service | Function |
| ------- | -------- |
| S3 Receiving Buckets | Worker reads tar files from depositor receiving buckets, extracting payload files and tag files to be copied to staging.
| S3 Staging Bucket | Worker copies payload files and tag files to from receiving bucket to staging bucket. Files go into staging with a UUID key, not their actual file name.
| Redis | Worker updates Redis file records to indicate files have been copied to staging.
| Registry | Source of WorkItem record describing work to be done.
| NSQ | Distributes WorkItem IDs to workers and tracks their status.

## Source Files

| Worker | Service | Files | Definition |
| ------ | ------- | ----- | ---------- |
| Staging Uploader | Ingest | [Task](https://github.com/APTrust/preservation-services/blob/master/ingest/staging_uploader.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/ingest_staging_uploader.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/ingest_staging_uploader/ingest_staging_uploader.go){target=_blank} | Copies files from a tarred bag in a receiving bucket to the ingest staging bucket. |
