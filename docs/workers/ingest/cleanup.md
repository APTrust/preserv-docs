# Cleanup

After an ingest has been successfully recorded, the cleanup worker does the following:

* Deletes all of the interim ingest data from Redis
* Deletes all of the temporary ingest files from the staging bucket
* Deletes the bag (the tar file) from the depositor's receiving bucket
* Marks the ingest WorkItem as complete in Registry

This is the only ingest work with no "next" queue. When it's done, it does not push the ingest WorkItem ID into another NSQ topic. It simply marks it as complete in the `ingest09_cleanup` topic, and that's the last NSQ hears of it.

!!! Note

    Cleanup does not run on ingests that fail due to transient errors. Those
    ingests can and should be requeued through the Registry when transient
    errors pass. (They are usually network errors or "service unavailable"
    errors.)


## Resources

This worker uses little CPU and memory. It may issue a lot of deletion requests to the staging bucket, but even with those, it uses little network I/O.

## External Services

| Service | Function |
| ------- | -------- |
| S3 Staging Bucket | Worker deletes all temp files related to this ingest from the staging bucket.
| S3 Receiving Buckets | Worker deletes the tar file uploaded by the depositor from the receiving bucket.
| Redis | Worker deletes all metadata related to this ingest from Redis.
| Registry | Source of WorkItem record describing work to be done. Worker marks the ingest WorkItem complete.
| NSQ | Worker marks the NSQ task complete.
