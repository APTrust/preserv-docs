# Anatomy of a Worker

Most workers consist of three components:

* A task definition file that implements task-specific business logic.
* A worker file that defines workflow policy including which NSQ channels to listen to, how to handle errors, and more.
* An app file that allows a worker to be compiled to a standalone application.

## Task Definition Files

The task definition file implements functions that the worker will perform on data. This file focuses on a single data-centric operation. For example, the [fixity checker](https://github.com/APTrust/preservation-services/blob/master/fixity/checker.go){target=_blank} calculates fixity on a file in preservation storage. The [bag validator](https://github.com/APTrust/preservation-services/blob/master/ingest/metadata_validator.go){target=_blank} validates a bag before ingest. The [file restorer](https://github.com/APTrust/preservation-services/blob/master/restoration/file_restorer.go){target=_blank} restores individual files.

All of these files implement a method called `Run()` to do the work. `Run()` returns an integer describing the number of tasks processed and a slice of [ProcessingError](https://github.com/APTrust/preservation-services/blob/6824fab65420f4ff8af30457e9e7d9352ce83f6b/models/service/errors.go#L11){target=_blank} objects describing what, if anything went wrong during processing.

## Worker Files

Worker files include a [Context](https://github.com/APTrust/preservation-services/blob/master/models/common/context.go){target=_blank} object that allows the worker to talk to NSQ, Redis, Registry and S3/Glacier/Wasabi.

The [Base Worker](https://github.com/APTrust/preservation-services/blob/master/workers/base.go){target=_blank} defines a number of methods that use the context object to perform tasks common to all workers, such as getting work items from the registry, registering a listener with NSQ, etc. The base worker also handles interrupt and kill signals from the operating system.

Workers also include a [Settings](https://github.com/APTrust/preservation-services/blob/master/workers/worker_settings.go){target=_blank} object that describes which NSQ topics it should read from and write to, how many times it should attempt its task, etc.

The worker source files tend to be slim because they don't implement much logic. They simply load code from an underlying task file, configure it with custom settings, and wrap it all into the worker framework that can talk to all of the external services (NSQ, Redis, Registry, and S3/Glacier/Wasabi).

For example, if you look at [workers/ingest_format_identifier.go](https://github.com/APTrust/preservation-services/blob/master/workers/ingest_format_identifier.go){target=_blank}, you'll see the code does the following:

1. Creates a new Context object providing clients for Redis, Registry, NSQ and S3.
2. Creates a settings object telling the worker which NSQ topic to read from, which topic to push to if processing completes successfully, how many times to attempt its task, how to handle fatal errors, how many items to keep in its internal work queue, etc.
3. Creates an underlying task handler, which can be any object that implements the `Run()` interface (aka a Runnable).
4. Creates a worker object with the required context, settings, and Runnable.
5. Tells the worker to start listening to NSQ. It will listen to the channel specified in the Settings.

All workers follow this pattern.

!!! Note

    The cron jobs `apt_queue`, `apt_queue_fixity` and `ingest_bucket_reader`
    don't have task files. Because they are designed to run and exit, rather
    than listen forever, their logic is implemented directly in the worker
    files.

## App Files

Preserv's [apps directory](https://github.com/APTrust/preservation-services/tree/master/apps){target=_blank} contains a directory for each worker. Inside each directory is a single file with a `main()` method, which is required by Go for building executables. (Go allows only one main method per directory, which is why the app files are all in separate directories.)

The [format identifier app](https://github.com/APTrust/preservation-services/blob/master/apps/ingest_format_identifier/ingest_format_identifier.go){target=_blank} shows the pattern for app files, which do the following:

1. Read options from the command line. These are optional, but allow us to override the following key settings that typically come in through the environment, the .env file, or AWS's parameter store:
    a. ChannelBufferSize - This is the number of items the worker should keep in its internal work queue.
    b. NumWorkers - The number of go routines the worker should run. For example, if this is set to 2, the worker will run 2 concurrent instances of its task handler. (Two concurrent Runnables.)
    c. MaxAttempts defines the maximum number of non-fatal errors a task can encounter before it quits. Tasks that fail with non-fatal errors are marked as failed in the Registry, with the WorkItem.NeedsAdminReview flag set to true. The APTrust admin can requeue these items later.
2. Create a worker with the specified options. Remember that the worker starts listening to its NSQ channel as soon as it is created.
3. Sets up a blocking channel for this worker to listen to. Nothing happens in this channel, but because the worker has to listen to it forever, it cannot exit until the operating system kills the worker with a SIGINT or SIGKILL. Without this, the worker would exit immediately. See the [Base Worker](https://github.com/APTrust/preservation-services/blob/master/workers/base.go){target=_blank} for info on how signals are handled.

## Source Files

| Worker | Service | Files | Definition |
| ------ | ------- | ----- | ---------- |
| Bucket Reader | Ingest | No Task File <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/ingest_bucket_reader.go){target=_blank} <br/>  [App](https://github.com/APTrust/preservation-services/blob/master/apps/ingest_bucket_reader/ingest_bucket_reader.go){target=_blank} | Cron job that scans depositor receiving buckets for new tar files to ingest. |
| Metadata Gatherer | Ingest | [Task](https://github.com/APTrust/preservation-services/blob/master/ingest/metadata_gatherer.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/ingest_pre_fetch.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/ingest_pre_fetch/ingest_pre_fetch.go){target=_blank} | Parses a bag's tag files, calculates checksums on bag contents, and copies manifests and tag files to the ingest staging bucket. |
| Bag Validator | Ingest | [Task](https://github.com/APTrust/preservation-services/blob/master/ingest/metadata_validator.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/ingest_validator.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/ingest_validator/ingest_validator.go){target=_blank} | Validates a bag before ingest. |
| Reingest Manager | Ingest | [Task](https://github.com/APTrust/preservation-services/blob/master/ingest/reingest_manager.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/reingest_manager.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/reingest_manager/reingest_manager.go){target=_blank} | Checks if a bag is being reingested and if so, applies special processing. |
| Staging Uploader | Ingest | [Task](https://github.com/APTrust/preservation-services/blob/master/ingest/staging_uploader.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/ingest_staging_uploader.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/ingest_staging_uploader/ingest_staging_uploader.go){target=_blank} | Copies files from a tarred bag in a receiving bucket to the ingest staging bucket. |
| Format Identifier | Ingest | [Task](https://github.com/APTrust/preservation-services/blob/master/ingest/format_identifier.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/ingest_format_identifier.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/ingest_format_identifier/ingest_format_identifier.go){target=_blank} | Identifies the format of files within a bag. |
| Preservation Uploader | Ingest | [Task](https://github.com/APTrust/preservation-services/blob/master/ingest/preservation_uploader.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/ingest_preservation_uploader.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/ingest_preservation_uploader/ingest_preservation_uploader.go){target=_blank} | Copies files to preservation storage. |
| Preservation Verifier | Ingest | [Task](https://github.com/APTrust/preservation-services/blob/master/ingest/preservation_verifier.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/ingest_preservation_verifier.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/ingest_preservation_verifier/ingest_preservation_verifier.go){target=_blank} | Verifies that files copied to preservation storage are actually there. |
| Ingest Recorder | Ingest | [Task](https://github.com/APTrust/preservation-services/blob/master/ingest/recorder.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/ingest_recorder.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/ingest_recorder/ingest_recorder.go){target=_blank} | Records all ingest data in Registry. |
| Cleanup | Ingest | [Task](https://github.com/APTrust/preservation-services/blob/master/ingest/cleanup.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/ingest_cleanup.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/ingest_cleanup/ingest_cleanup.go){target=_blank} | Cleans up all of the temporary resources created during the ingest process and deletes ingested bags from receiving buckets. |
| APT Queue Fixity | Fixity | No Task File <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/queue_fixity.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/apt_queue_fixity/apt_queue_fixity.go){target=_blank} | Queues files for fixity checks. |
| Fixity Checker | Fixity | [Task](https://github.com/APTrust/preservation-services/blob/master/fixity/checker.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/fixity_checker.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/apt_fixity/apt_fixity.go){target=_blank} | Checks fixity on files in preservation storage. (S3 and Wasabi only. Does not check Glacier files.) |
| Glacier Restorer | Restoration | [Task](https://github.com/APTrust/preservation-services/blob/master/restoration/glacier_restorer.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/glacier_restorer.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/glacier_restorer/glacier_restorer.go){target=_blank} | Moves files from Glacier into S3 for restoration. |
| File Restorer | Restoration | [Task](https://github.com/APTrust/preservation-services/blob/master/restoration/file_restorer.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/file_restorer.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/file_restorer/file_restorer.go){target=_blank} | Restores individual files. |
| Object Restorer | Restoration | [Task](https://github.com/APTrust/preservation-services/blob/master/restoration/bag_restorer.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/bag_restorer.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/bag_restorer/bag_restorer.go){target=_blank} | Restores entire bags (intellectual objects). |
| Deletion Manager | Deletion | [Task](https://github.com/APTrust/preservation-services/blob/master/deletion/manager.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/deleter.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/apt_delete/apt_delete.go){target=_blank} | Deletes files and objects from preservation storage. |
| APT Queue | Deletion and Restoration | No Task File <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/apt_queue.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/apt_queue/apt_queue.go){target=_blank} | This cron job periodically scans Registry for restoration and deletion requests that have not been queued in NSQ. |
