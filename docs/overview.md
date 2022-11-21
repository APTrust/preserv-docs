# System Overview

Preservation services consists of the following components:

* A set of Docker containers, each running a microservice to handle various aspects of the ingest, restoration, deletion and fixity processes.
* NSQ - a queue service for orchestrating work items.
* Redis/Elasticache - for sharing interim processing data between workers.
* S3, Glacier and Wasabi for storage
* The Registry REST API for storing and retrieving persistent metadata about the content of our preservation repository.

While other parts of this documentation describe the components in detail, this page provides a graphical overview of the system's components, where they live, and how they communicate.

# The High Level

The components of the system are divided into three privilege zones, with privileges enforced by IAM roles and policies.

Depositors have access to the leftmost zone in the diagram below. They can upload files into their own receiving buckets for ingest, and download them from their own restoration buckets for retrieval. Depositors can access only their own buckets, no one elses.

The rightmost zone is preservation storage. No one can access this except the IAM account of the Go workers, which has full access, and APTrust admins, who have read-only access.

The Go service workers move items from depositor receiving buckets on the left into long-term preservation storage buckets on the right. They also move items from long-term storage back to depositor restoration buckets. The workers keep temporary housekeeping data about items in process in Redis/Elasticache, and store permanent data in the Registry. All of the working components of the system--all services using CPU and memory--live in the middle zone.

Depositors cannot access anything in the middle zone. APTrust administrators can, generally through the AWS console and CloudFormation templates.

![High level diagram showing zones and components](img/architecture/Preserv.drawio.png)

# Ingest

During the ingest process, depositors upload bags (tar files) to their receiving buckets. A cron scans for new bags, creating a WorkItem in the Registry for each one, and then putting that WorkItem's ID into NSQ for the Ingest workers to pick up.

Ingest workers perform a number of functions, including bag validation, format identification, and more. Then they move files into preservation storage and record the results of their work in the Registry.

![](img/architecture/Preserv-Ingest-01.drawio.png)

!!! Note
    In the diagrams below, all ingest workers read to and write from
    both Redis and NSQ. Arrows are omitted to minimize clutter.

## NSQ

NSQ keeps track of which tasks need to go to which workers. For Ingest, a cron job running in the `apt_queue` container creates and queues new WorkItems for each new bag that shows up in a receiving bucket.

For deletion and restoration, users make requests through the Registry Web UI, and Registry pushes the WorkItem IDs into NSQ's deletion and restoration topics.

For fixity checks, a cron job running in `apt_queue_fixity` queries Registry for files in S3 or Wasabi storage that have not had a fixity check in 90 days. It pushes the IDs of those files into NSQ, usually in batches of 2,500-3,000 every half hour.

![](img/architecture/Preserv-Ingest-01-NSQ.drawio.png)

## Redis/Elasticache

Redis stores info about what work has already been done on any task in the ingest process. New workers can pick up partially completed tasks and pick up work where the last worker left off. The housekeeping data in Redis ensures two things:

1. that no work is skipped or left incomplete
2. that completed work is not unnecessarily repeated

![](img/architecture/Preserv-Ingest-01-Redis.drawio.png)

## The Pre-Fetch Worker

The pre-fetch worker streams a bag from a depositor's receiving bucket through a series of functions to do the following:

1. Gather a list of files in the bag.
2. Calculated fixity values on those files.
3. Extract tag files, manifests, and tag manifests to an S3 staging bucket for later processing.

![](img/architecture/Preserv-Ingest-02-Pre-Fetch.drawio.png)

## The Validation Worker

The validation worker ensures a bag is valid according to its BagIt profile. APTrust currently supports APTrust and BTR (Beyond the Repository) BagIt formats.

A bag is valid if it contains all of the payload files mentioned in the manifests, and if the checksums of those files match the checksums in the manifests. The bag must also contain a set of BagIt standard tags with valid values, plus whatever tags are required by the specified BagIt profile.

Note that profiles also specify which checksum algorithms and required and which others are permitted. The validator enforces those rules as well. If a bag is incomplete, if it contains extraneous payload files not mentioned in the manifests, or if it is missing required tags or required checksum algoriths, it will fail validation, the system will not ingest it, and processing stops here.

The validation worker tends to complete its work quickly, generally in a fraction of a second. It simply analyzes all of the metadata stored in Redis by the pre-fetch worker, which as already done the hard work of inventorying contents and calculating checksums.

![](img/architecture/Preserv-Ingest-03-Validate.drawio.png)

## Reingest Check

The reingest checker asks Registry if this bag has ever been ingested before. Bags have unique identifiers composed of the identifier of the owning instituion, followed by a slash and the name of the bag, minus the .tar extension. For example, if the University of Virginia uploads a bag called `photos.tar`, the bag's Intellectual Object Identifier becomes `virginia.edu/photos`.

If the Registry has an active (non-deleted) record of an object whose identifier matches the one we're currently ingesting, it checks each file in the new bag against records in the registry. If the file exists in Registry, the worker compares the Registry checksum to the checksum calculated by the pre-fetch worker.

If the checksum has changed, or if the file does not exist in Registry, the worker flags the file as needing to be saved to preservation storage. If Registry already has a record of this file and the checksum has not changed, the file will be flagged as not needing to be copied to preservation.

The reingest checker prevents us from storing duplicate copies of existing files under new UUIDs. (Everything in preservation storage is stored with a UUID key, not its original file name.)

![](img/architecture/Preserv-Ingest-03-ReingestCheck.drawio.png)

## Staging Uploader

The staging uploader unpacks bags from depositor ingest buckets and stores their individual files in an S3 staging bucket. Files in the staging bucket are stored with a key composed of WorkItem ID and file UUID. E.g. `34546/ddeff40b-f995-42f0-b002-389cf331193d`.

Subsequent workers that need to access these files can retrieve them from this staging bucket. S3 works well for write-once/read-multiple workloads. Because it's a network resource, not tied to local block storage, its contents are available to all of our ingest workers.

![](img/architecture/Preserv-Ingest-04-Staging.drawio.png)

## Format Identifier

The format identifier streams files from the staging bucket through a set of algorithms that compare the file's bits to known signatures in the PRONOM registry. It updates each file record in Redis with the file's mime type.

![](img/architecture/Preserv-Ingest-06-FormatID.drawio.png)

## Preservation Uploader

The preservation upload copies files from the staging bucket to preservation storage.

![](img/architecture/Preserv-Ingest-07-Storage.drawio.png)

## Preservation Verifier

The preservation verifier performs a HEAD request on every item copied into preservation storage to retrieve its metadata. Where possible, it compares each file's etag in preservation storage to its md5 checksum to ensure integrity. Etag comparison isn't possible on larger files, so it compares the file size in preservation with its know file size.

The verifier is basically a sanity check to provide basic assurance that files were successfully copied to preservation.

![](img/architecture/Preserv-Ingest-08-StorageValidation.drawio.png)

## Recorder

The recorder records metadata about the ingest in the Registry, including info about the Intellectual Object and all of its files. For each file, it records checksums, storage records and a set of ingest Premis events. It also records Premis events for the object.

![](img/architecture/Preserv-Ingest-09-RecordMetadata.drawio.png)

## Cleanup

The cleanup worker deletes the original bag from the depositor's receiving bucket, then deletes all interim processing data from the S3 staging bucket and Redis. Finally, it marks the ingest as complete in the Registry and in NSQ.

![](img/architecture/Preserv-Ingest-10-Cleanup.drawio.png)
