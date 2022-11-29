# The Preservation Uploader

The preservation uploader copies files from the S3 staging bucket to long-term preservation storage, which may be another S3 bucket, a Glacier vault, or a Wasabi bucket. (See [Preservation Buckets](/components/s3/#preservation-buckets) for details on where they are.)

When copying to the main preservation bucket in Virginia, the uploader uses AWS's bucket-to-bucket copying. When copying to Wasabi or to any region outside of us-east-1, the uploader streams the file through the Docker container to preservation storage. Ideally, it would use bucket-to-bucket copying for other AWS regions, but bucket-to-bucket is unacceptably slow for inter-region copies.

This worker creates a StorageRecord for each copy of each file it moves into preservation storage. The StorageRecord includes information about where and when the file was stored. These records are attached to the file record and stored in Redis. They will eventually go into the registry.

## Resources

The preservation uploader can use substantial amounts of memory and network I/O.

## External Services

This worker talks to:

* S3 Staging Bucket
* Preservation Buckets
* Redis
* Registry
* NSQ
