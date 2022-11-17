# Registry

As far as Preserv is concerned, Registry is a REST API containing the following info.

## WorkItems

Registry workers get WorkItem IDs from NSQ topics, and then look up those work items through the Registry REST API. Each worker determines whether it should work on the item, based on data it contains.

Workers may reject items if they see any of the following info in the WorkItem record:

* The item has a non-empty node and pid, indicating it is already in process by another worker.
* The item's Action or Stage does not match the Action and Stage the worker should be working on.
* The item is in some final state, such as `Cleanup/Succeeded` or `Cancelled`.

In all of these cases, the worker will drop the item without doing any work.

If a Preserv worker chooses to work on an item, it updates the WorkItem record in the Registry with its Node and PID, and changes the status from `Pending` to `Started`.

### Fixity Check Work Items

Unlike all other queue topics, the fixity check topic does not contain WorkItem IDs. It contains GenericFile IDs. These are the IDs of files that need a scheduled fixity check.

The fixity work retrieves the Generic File record from Registry, ensures the file has not been checked in 90 days, then runs its check. When it's done, it creates a Premis Event describing the result of the check.

## Objects, Files, Storage Records, Checksums and Events

During ingest, the re-ingest checker pulls Intellectual Object and Generic Files from Registry to see if an object or its component files has ever been ingested before. See the [Reingest Manager](../workers/ingest/reingest-manager.md) page for details on that.

Near the end of the ingest process, the [Recorder](../workers/ingest/recorder.md) saves data about the ingest to the Registry, including:

* The Intellectual Object record.
* One Generic File record for each file in the object.
* One Storage Record for each copy of the Generic File. These point to where the file is stored in preservation. Most storage options result in a single storage record. The `Standard` option results in two records: one for S3 in Virginia and one for Glacier in Oregon.
* Checksums for each file ingested. The old system (Exchange) recorded only md5 and sha256 checksums. The new system records md5, sha1, sha256 and sha512 checksums for each file. Note that we only check the sha256 during scheduled fixity checks.
* A set of Premis Events for each file, including:
    * One fixity check indicating the file's initial fixity matched what's in the bag's manifest.
    * One identifier assignment describing the file's URL in preservation storage.
    * A second identifier assignment, if the file is in Standard storage, describing the URL of the file's second copy in Oregon.
    * One ingestion event, indicating the file has been ingested into preservation storage.
    * One replication event, if the file is in Standard storage, indicating a second copy of the file was saved to Glacier/Oregon.
    * Four message digest calculations: one each for md5, sha1, sha256 and sha512
* A set of Premis Events for the object itself, including:
    * Ingestion of the object into preservation storage.
    * Creation of the object record in Registry.
    * Identifier assignment, indicating the assignment of this object's unique identifier in Registry.
    * Access assignment, based on the value of the Access tag in the bag's aptrust-info.txt file.
