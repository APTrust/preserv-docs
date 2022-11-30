# The Metadata Gatherer

The metadata gatherer, ingest_pre_fetch, streams a bag from a depositor's receiving bucket though a series of functions to do the following:

* Calculate checksums on all files in the bag.
* Parse the bag's tag files.
* Extract the bag's tag files and manifests.
* Collect general metada about the bag, including it's name, size, number of files, and owning institution.

The worker stores the tag files and manifests in the S3 staging bucket. In production, that would be bucket `aptrust.prod.staging`. All of these files go into a folder under the WorkItem ID. For example, the pre-fetch worker would produce the following set of staging files for WorkItem 6388:

```
aptrust.prod.staging/6388/aptrust-info.txt
aptrust.prod.staging/6388/bag-info.txt
aptrust.prod.staging/6388/manifest-md5.txt
aptrust.prod.staging/6388/manifest-sha256.txt
aptrust.prod.staging/6388/tagmanifest-md5.txt
aptrust.prod.staging/6388/tagmanifest-sha256.txt
```

The worker also stores all of the essential medata it gathered in Redis. This includes JSON records for every file in the bag, with each JSON record recording, among other things, the file's path and checksums.

The next worker, the validator, will examine this data to ensure the bag is valid and can be ingested.

For details on what the Redis data looks like, see the section on [Querying Redis](../../../components/redis/#querying-redis), which includes sample records.

## Resource Usage

This worker uses a substantial amount of network bandwidth (streaming bags from receiving buckets) and CPU (for calculating multiple checksums on files).

## External Services

| Service | Function |
| ------- | -------- |
| S3 Receiving Buckets | Worker reads tar files from depositor receiving buckets.
| S3 Staging Bucket | Worker copies manifests and tag files (but not other files) to the staging bucket for later access by the bag validator.
| Redis | Worker saves metadata about the bag and all of its files in JSON format to Redis, where all subsequent workers can access it.
| Registry | Source of WorkItem record describing work to be done.
| NSQ | Distributes WorkItem IDs to workers and tracks their status.
