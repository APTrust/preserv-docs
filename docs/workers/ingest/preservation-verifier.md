# The Preservation Verifier

The preservation verifier checks that all of the files copied by the preservation uploader are actually present in the preservation storage buckets. It issues a HEAD request for each file and ensures that the size matches. When possible, it also ensures that the etag matches the file's md5 checksum. (This is only possible for smaller files, not for large, mutli-part uploads.)

If any check fails, the verifier marks the ingest as failed, sets a note about the missing/incorrect file in the note field of the WorkItem, and sets the WorkItem's `NeedsAdminReview` flag to true.

## Resources

Though this worker may issue a number of S3 requests, it does not use much network bandwidth because HEAD requests tend to return about 1 kb of data. The worker uses little CPU and memory, and tends to finish quickly.

## External Services

This worker talks to:

* S3 preservation buckets
* Redis
* Registry
* NSQ
