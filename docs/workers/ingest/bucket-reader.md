# The Bucket Reader

The bucket reader, ingest_bucket_reader, runs as a cron job inside its own container. It scans all receiving buckets belonging to all depositors for new items. (Items here means tar files containing BagIt bags for ingest.)

When it finds items in receiving buckets, it does the following:

1. Checks the Registry to see if a WorkItem exists with action `Ingest` for the tar file. It matches based on the name and etag of the tar file and the ID of the institution that owns the bucket.
2. If a matching WorkItem exists, ingest_bucket_reader does nothing. If there is no matching item...
3. ingest_bucket_reader creates an Ingest WorkItem for the bag.
4. ingest_bucket_reader adds the WorkItem ID to NSQ's `ingest_01_prefetch` topic.

ingest_bucket_reader logs what it does with every item it encounters, and why. If you want to know what it's doing, check the container logs in CloudWatch.

## Why cron? Why not use S3 events and lambdas?

When APTrust first launched in 2014, AWS lambdas didn't exist. Even if they had, we had a bigger problem. The first version of Registry, called Fluctus, was so unreliable, there was no guarantee that it would even be running when S3 events were fired. There was a high likelihood of events being missed.

Our cron jobs scan everything each time they run. If the message receiver (Registry) missed cron's messages for any reason, it will get them again in a few minutes.

In addition, APTrust's original design requirements explicitly stated that the system must be able to run on any Linux box, anywhere, without relying on any vendor-specific services other than S3 and Glacier.

Cron jobs were portable and reliable then, and they still are today. We continue to use them because they work.

To put it another way, why do extra work to lock yourself into vendor-specific services, adding another layer of complexity and incurring additional operating costs, when you can do no work and have none of those problems? If you can make a business case for that, there's a position waiting for you at Accenture.

## External Services

| Service | Function |
| ------- | -------- |
| S3 Receiving Buckets | The reader scans depositor receiving buckets for new bags (tar files) to be ingested.
| Registry | The reader creates new WorkItems here for each bag awaiting ingest.
| NSQ | The reader adds WorkItem IDs for bags awaiting ingest to NSQ's `ingest01_pre_fetch` topic.
