# NSQ

NSQ is the queue service that tells all our workers what tasks need to be completed. This includes ingest, restoration, deletion and fixity workers.

NSQ includes topics and channels. A topic holds a list of tasks that need to be completed. Each topic can have one or more channels. Workers subscribe to channels to get their work.

!!! Note

    NSQ allows multiple channels per topic because in certain environments
    mutiple workers need to take action when an item appears in a queue.
    In Preserv, we use only one channel per topic.

In Preserv, each topic has only one channel, and workers subscribe to channels as described in the table below. The `source` column describes how items get into this queue. The `content` column describes what's in the queue message. When the content is a WorkItem ID, the worker pulls a copy of the WorkItem from the Registry, marks it as "in progress" and begins its work. When the the content is a GenericFile ID, the worker retrieves the GenericFile record from the Registry, runs a fixity check, and records the outcome in a Premis Event withouth ever creating a WorkItem.

Topic | Worker | Content|  Purpose | Source
----- | ------ | ------- | ------- | ------
delete\_item | [apt\_delete](/workers/deletion/) | WorkItem ID | Deletes files and/or objects from preservation storage and records deletion Premis Event. | Deletions are pushed into the queue by the Registry when they're approved by an institutional admin. The apt\_queue service runs periodically to catch any deletion and restoration requests that may not have made it from Registry to NSQ due to network errors.
fixity\_check | apt\_fixity | GenericFile ID | Tells apt\_fixity which files to check. | The apt\_queue_fixity process pushes about 2500 files into this queue every 20-30 minutes. The exact number and frequency are defined by params `QUEUE_FIXITY_INTERVAL` and `MAX_FIXITY_ITEMS_PER_RUN`, both of which are in AWS Parameter Store.
ingest01\_ prefetch | [ingest\ _pre\_ fetch](/workers/ingest/pre-fetch/) | WorkItem ID | Scans new bags in depositors' receiving buckets and collects metadata for bag validation. | The ingest\_bucket\_reader, a cron job running in its own container, checks the buckets every few minutes. When it finds new bags with no associated WorkItem, it creates a new Ingest WorkItem in the Registry and pushes the WorkItem ID into this queue.
ingest02\ _bag\_ validation | [ingest\_ validator](/workers/ingest/validator/) | WorkItem ID | Validates a bag to ensure all files are present, all checksums match, no extraneous payload files exist, and the bag conforms to the BagIt profile noted in bag-info.txt. | The ingest\_pre\_fetch worker pushes WorkItem IDs into this queue for each item it successfully processes.
ingest03\_ reingest\_ check | [reingest\_ manager](/workers/ingest/reingest-manager/) | WorkItem ID | Checks files in the bag against known files in the Registry to see if any of them are re-ingests. See the [reingest manager page](/workers/ingest/reingest-manager/) for more info. | Filled by ingest\_validator.
ingest04\_ staging | [ingest\_ staging\_ uploader](/workers/ingest/staging-uploader/) | WorkItem ID | Unpacks tarred bags from depositor receiving buckets and uploads the individual files to an S3 staging bucket so subsequent workers can process them. | Filled by reingest\_manager.
ingest05\_ format\_ identification | [ingest\_ format\_ identifier](/workers/ingest/format-identifier/) | WorkItem ID | Runs individual files from the staging bucket through the Siegfried file identifier, which uses the PRONOM registry under the hood to identify file formats. | Filled by ingest\_ staging\_ uploader.
ingest06\_ storage | [ingest\_ preservation\_ uploader](/workers/ingest/preservation-uploader/) | WorkItem ID | Copies individual files from the S3 staging bucket to preservation storage in S3, Glacier, and/or Wasabi. | Filled by ingest\_ format\_ identifier.
ingest07\_ storage\ _validation | [ingest\_ preservation\_ verifier](/workers/ingest/preservation-verifier/) | WorkItem ID | Verifies that items copied to preservation storage actually got there by performing a HEAD request and ensuring file size matches. | Filled by ingest\_ preservation\_ uploader.
ingest08\_ record | [ingest\_ recorder](/workers/ingest/recorder/) | WorkItem ID | Records the results of an ingest in Registry. This includes the object record, file records, storage records, checksums, and premis events. | Filled by ingest\_ preservation\_ verifier.
ingest09\_ cleanup | [ingest\_ cleanup](/workers/ingest/cleanup/) | WorkItem ID | Deletes interim processing data, including the tarred bag in the depositor's receiving bucket, the individual files in the staging bucket, and the processing metadata in Redis/Elasticache. | Filled by ingest\_recorder.
restore\_ file | [file\_ restorer](/workers/restoration/file-restorer/) | WorkItem ID | Restores individual files by moving them from preservation storage to the depositor's restoration bucket. | Filled by Registry when a user clicks the `Restore File` button, with backup from `apt_queue` in case Registry queueing fails. This topic can also be filled by restore\_glacier worker when it determines that a Glacier file has been moved into an S3 bucket for retrieval.
restore\_ glacier | [glacier\_ restorer](/workers/restoration/glacier-restorer/) | WorkItem ID | Moves files from Glacier and Glacier Deep Archive into an S3 bucket so they can be restored to depositors. | Filled by Registry when a user clicks the `Restore File` button for an object in Glacier or Glacier Deep Archive. Registry queueing fails, `apt_queue` will find the item and queue it.
restore\_ object | [bag\_ restorer](/workers/restoration/bag-restorer/) | WorkItem ID | Restores entire objects (bags) to depositors by collecting all files, bagging them, validating the bag, and pushing it into the depositor's restoration bucket. | Filled by Registry when a user clicks the `Restore File` button, with backup from `apt_queue` in case Registry queueing fails. This topic can also be filled by restore\_glacier worker when it determines that all of a bag's Glacier files have been moved into an S3 bucket for retrieval.

## Duplicate Tasks in NSQ

Under [Message Delivery Guarantees](https://nsq.io/overview/design.html#message-delivery-guarantees){target=_blank} NSQ documentation says:

> NSQ guarantees that a message will be delivered at least once, though duplicate messages are possible. Consumers should expect this and de-dupe or perform idempotent operations.

It's fairly common to get duplicate messages in most of the NSQ topics. This generally happens in two ways:

1. In the fixity topic, `apt_queue_fixity` finds 2500 files in the Registry that have not had a fixity check in 90 days. It queues the GenericFile ID of those files in NSQ's `fixity_check` topic. Occasionally, `apt_fixity` can't get through all 2500 files before the next run of `apt_queue_fixity`, which then queues incomplete the GenericFile IDs again.

2. In all other topics, NSQ may not hear back from a worker before NSQ's internal timeout. NSQ thinks the worker is stuck or dead, so it requeues the WorkItem ID itself. This is most likely to happen when long-running operations like `ingest_pre_fetch`, `ingest_format_identifier` and `ingest_preservation_uploader` are dealing with huge files.

The solutions to these issues are:

1. The `apt_fixity` checks the LastFixity timestamp on each GenericFile before it actually starts work. If the file was just checked, it tells NSQ the task is complete and logs a message saying it's skipping that file because it was recently checked.

2. All other workers fetch a copy of the WorkItem from the Registry before they go to work. If the WorkItem Node is non-empty, the process knows some worker is already on it, and it skips the task. Workers also check a number of other properties on the WorkItem to ensure the item is in the right stage of processing, has not completed or been cancelled, has the Retry flag set to true, etc.

For more info, search the code repo for `ShouldSkipThis`. You'll find various methods in various packages, since the logic for what's to be skipped differs according to what's being done. All ingest workers share the same skip logic. Bag restoration, file restoration, Glacier restoration and deletion have their own logic, but regardless of the worker, the function is always called `ShouldSkipThis`

## Queue Settings in AWS Parameter Store

Each worker pulls three performance tuning parameters from Parameter Store. The settings are listed below. Descriptions of the settings appear below.

Queue Topic | Settings |
----------- | -------- |
delete_item | APT_DELETE_BUFFER_SIZE, APT_DELETE_WORKERS, APT_DELETE_MAX_ATTEMPTS
fixity_check | APT_FIXITY_BUFFER_SIZE, APT_FIXITY_WORKERS, APT_FIXITY_MAX_ATTEMPTS
ingest01_prefetch | INGEST_PRE_FETCH_BUFFER_SIZE, INGEST_PRE_FETCH_WORKERS, INGEST_PRE_FETCH_MAX_ATTEMPTS
ingest02_bag_validation | INGEST_VALIDATOR_BUFFER_SIZE, INGEST_VALIDATOR_WORKERS, INGEST_VALIDATOR_MAX_ATTEMPTS
ingest03_reingest_check | REINGEST_MANAGER_BUFFER_SIZE, REINGEST_MANAGER_WORKERS, REINGEST_MANAGER_MAX_ATTEMPTS
ingest04_staging | INGEST_STAGING_UPLOADER_BUFFER_SIZE, INGEST_STAGING_UPLOADER_WORKERS, STAGING_UPLOADER_MAX_ATTEMPTS
ingest05_format_identification | INGEST_FORMAT_IDENTIFIER_WORKERS, INGEST_FORMAT_IDENTIFIER_BUFFER_SIZE, INGEST_FORMAT_IDENTIFIER_MAX_ATTEMPTS
ingest06_storage| INGEST_PRESERVATION_UPLOADER_BUFFER_SIZE, INGEST_PRESERVATION_UPLOADER_WORKERS, INGEST_PRESERVATION_UPLOADER_MAX_ATTEMPTS
ingest07_storage_validation | INGEST_PRESERVATION_VERIFIER_BUFFER_SIZE, INGEST_PRESERVATION_VERIFIER_WORKERS, INGEST_PRESERVATION_VERIFIER_MAX_ATTEMPTS
ingest08_record | INGEST_RECORDER_BUFFER_SIZE, INGEST_RECORDER_WORKERS, INGEST_RECORDER_MAX_ATTEMPTS
ingest09_cleanup | INGEST_CLEANUP_BUFFER_SIZE, INGEST_CLEANUP_WORKERS, INGEST_CLEANUP_MAX_ATTEMPTS
restore_file | FILE_RESTORER_BUFFER_SIZE, FILE_RESTORER_WORKERS, FILE_RESTORER_MAX_ATTEMPTS
restore_glacier | GLACIER_RESTORER_BUFFER_SIZE, GLACIER_RESTORER_WORKERS, GLACIER_RESTORER_MAX_ATTEMPTS
restore_object | BAG_RESTORER_BUFFER_SIZE, BAG_RESTORER_WORKERS, BAG_RESTORER_MAX_ATTEMPTS

### Workers Setting

Params ending in `WORKERS` describe the number of go routines a worker should use to do its work. A setting of 3 indicates 3 go routines, which means the worker can process three NSQ WorkItems simultaneously. (Think of a Java or C++ app running three independent threads at all times, with each thread capable of handing an NSQ task from stat to finish.)

Because our workers run in very small containers with >512 MB of RAM, we tend to limit the number of workers to 2, or in some cases, such as the format identifier, to 1. (The format identifier uses a lot of memory, and more than one concurrent go routine can cause it to crash with an out-of-memory exception.)

When running in larger containers, you can increase the number of workers. Note, however, that `ingest_preservation_uploader` can also use a lot of memory and can crash if you set the number of workers too high.

!!! important

    It's generally not a good idea to set the number of workers for
    ingest_recorder above 2. This worker issues a huge number of
    requests to the Registry at the end of the ingest process. Too many
    recorders can make the Registry very slow, or can cause it to
    scale out to more containers than we really need to run.

    It's better (and likely just as fast) to keep these items queued
    in the record topic than to have hundreds of requests stuck in
    Registry's TCP buffers.

### Buffer Size Setting

Params ending in `BUFFER_SIZE` describe the number of NSQ messages to hold in each worker's internal queue. This setting has some interesting ramifications for scaling.

Let's say you set `BUFFER_SIZE` to 10 for the `ingest_staging_uploader`, and you also set the number of workers for `ingest_staging_uploader` to 2. Each of the ingest validator's two go routines will tell NSQ it's ready to accept 10 items. Assuming NSQ has that many items, it will give 20 tasks to `ingest_staging_uploader` all at once, and **it will wait until the each go routine has finished its set of 10 before it hands that go routine 10 more.**

This last point is important for scaling. Let's say a worker gets 20 items from NSQ and its CPU and RAM usage go straight to 100%. This will trigger our ECS auto-scaler to add a new `ingest_staging_uploader` container.

However, the first container will not relinquish the batch of 20 files it's working on, and it's possible that the new container will have nothing to work on at all. In this case, container 1 may continue to max out all its resources for several hours, while container 2 remains idle.

For this reason, we tend to set `BUFFER_SIZE` to a low number (3-5) for our long-running, resource-intensive workers. The following workers are resource-intensive and/or long-running, and should have relatively small buffer sizes:

Worker | Heavily Used Resource
------ | ---------------------
apt_fixity | High network I/O when fetching files from S3. High CPU usage when calculating checksums.
bag_restorer | High network I/O when fetching files from S3. High CPU usage when calculating checksums.
file_restorer | High network I/O when fetching files from S3. High CPU usage when calculating checksums.
ingest_format_identifier | High network I/O and CPU. Extremely high memory.
ingest_pre_fetch | High network I/O when fetching files from S3. High CPU usage when calculating checksums.
ingest_preservation_uploader | High network I/O. When uploading very large files, high memory usage for upload buffers.
ingest_staging_uploader | High network I/O. When uploading very large files, high memory usage for upload buffers.

The remaining workers tend to finish to their tasks quickly, often in seconds, so they can tolerate higher buffer sizes.

### Max Attempts Setting

The `MAX_ATTEMPTS` settings describe how many times a worker should retry an NSQ task before giving up due to transient errors. Transient errors almost always fall into two categories:

1. Network problems, such as `timeout` or `connection reset by peer`
2. Unavailable remote services, such S3, Glacier, Wasabi, Redis, NSQ or Registry responding with `HTTP 503 (Service Unavailable)` or not responding at all.

Of the errors above, #1 accounts for more than 99% of cases.

When a task fails due to transient errors, the worker requeues it. After requeuing it `MAX_ATTEMPS` times, the worker stops trying to complete the task and does the following:

1. Marks the taks as failed in NSQ.
2. Logs specific information about why it's quitting.
3. Updates the WorkItem in Registry, setting:
    1. Retry = false
    2. NeedsAdminReview = true
    3. Note = Detailed information about what happened and why the worker won't complete the task.

An APTrust admin can easily find these items in the Registry by search for WorkItems where NeedsAdminReview = true. The admin can then requeue them if he/she knows the underlying issue is resolved.

When these issues do occur, they are almost always network errors and it makes no sense to requeue this items until we know the network has recovered.

## Fatal Errors

The only fatal errors that will cause workers to permenantly give up on an item are:

* Invalid bags. These occasinally appear in depositors' receiving buckets. We can't process them, so we don't. Invalid bags may remain in the receiving buckets for up to 14 days, so we and the depositors can examine them. Their interim processing data may persist in Redis as well, for diagnostic/forensic purposes. APTrust admins must manually delete the Redis data.
* Files missing from preservation. This has never happened, but if it were to happen, restoration and deletion processes would log a fatal error and set the NeedsAdminReview flag on the appropriate WorkItem. The APTrust admin will have to manually investigate the issue.
* Invalid restoration package. This would happen if the object restorer reassembled a bag for restoration and then found the bag was invalid. This has never happened. Once again, this would require a manual investigation by an APTrust admin.

## NSQ Admin Functions

All NSQ admin features are available to APTrust admins through the Registry UI. Features include starting, pausing, deleting and emptying topics and channels.

!!! important

    The most graceful way to stop all processing in the system is to pause
    all of the topics and channels. The workers will complete the items in
    their internal go routine queues and will then sit idle.

    The Registry UI includes a feature to pause and unpause all topics
    and/or channels at once. When paused, topics will continue to queue
    new tasks, but will not pass those tasks out to workers. When you
    unpause the topics and channels, workers will resume work immediately.

## Manually Requeuing

The Registry UI allows APTrust admins to manually requeue any item that has not completed processing. (I.e. You can requeue a completed ingest, restoration, or deletion. There's no point.)

To requeue an item, go to it's WorkItem detail page and click the Requeue button. You should generally requeue to the last attempted stage of processing, which is pre-selected in the UI. The APTrust admin may requeue to any earlier stage of processing if he/she thinks that's appropriate.
