# apt_pre_fetch: The Metadata Gatherer

The apt\_pre\_fetch streams a bag from a depositor's receiving bucket though a series of functions to do the following:

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

This worker talks to:

* S3 receiving buckets
* S3 staging bucket
* Redis
* Registry
* NSQ
