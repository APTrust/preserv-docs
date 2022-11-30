# The Preservation Uploader

The preservation uploader copies files from the S3 staging bucket to long-term preservation storage, which may be another S3 bucket, a Glacier vault, or a Wasabi bucket. (See [Preservation Buckets](/components/s3/#preservation-buckets) for details on where they are.)

When copying to the main preservation bucket in Virginia, the uploader uses AWS's bucket-to-bucket copying. When copying to Wasabi or to any region outside of us-east-1, the uploader streams the file through the Docker container to preservation storage. Ideally, it would use bucket-to-bucket copying for other AWS regions, but bucket-to-bucket is unacceptably slow for inter-region copies.

This worker creates a StorageRecord for each copy of each file it moves into preservation storage. The StorageRecord includes information about where and when the file was stored. These records are attached to the file record and stored in Redis. They will eventually go into the registry.

## Resources

The preservation uploader can use substantial amounts of memory and network I/O.

## External Services

| Service | Function |
| ------- | -------- |
| S3 Staging Bucket | Worker copies files from staging to long-term storage.
| Preservation Buckets | Worker copies files from staging to long-term storage. Preservation buckets may include S3, Glacier, Glacier Deep Archive, and Wasabi.
| Redis | Worker adds storage records to each file record in Redis. The storage record describes which preservation bucket(s) the file was copied to and when. For standard storage, each file ends up with two storage records, one for S3/Virginia and one for Glacier/Oregon. All other storage options result in a single storage record.
| Registry | Source of WorkItem record describing work to be done.
| NSQ | Distributes WorkItem IDs to workers and tracks their status.
