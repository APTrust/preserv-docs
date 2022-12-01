# The Reingest Manager

The reingest manager checks to see whether a bag has ever been ingested before. All bags have a unique intellectual object identifier in the format institution_identifier/bag_name. When we're ingesting a bag from the University of Virginia's receiving bucket called photos.tar, the intellectual object identifier for that bag will be virginia.edu/photos.

The reingest manager asks the Registry if an active (non-deleted) object with that identifier already exists. If Registry has no active record of this object, all files in the bag will be saved to preservation storage under a new UUID.

If Registry does have a record of the bag, the reingest manager then compares the names and checksums of all the files in the new tarred bag to those in Registry.

* Files not in the Registry will be saved to preservation storage under a new UUID.
* Files in the Registry that have changed in the new tarred bag (i.e. those whose checksums differ from the Registry checksums) will be saved to preservation storage under Registry's existing UUID. We do not want two different versions of the same file saved under different UUIDs in storage because they cause confusion and add cost. We want only one authoritative version.
* Files in the bag whose checksums match Registry checksums will be marked as `NeedsSave = false` and will not be copied to preservation storage by the Preservation Uploader.

## Resources

The reingest manager typically uses very little CPU, memory and network I/O. Most bags are not reingests. The manager issues a single API call to the Registry's objects endpoint, learns the object is new, and its work is done.

If the Registry does have a record of the object, the reingest manager pulls file metadata records one by one from Redis, then pulls corresponding file records one by one from Registry and compares their checksums.

In typical cases, this results in a dozen or so calls to each service. In very rare cases (once or twice a year), it may lead to tens of thousands of calls to each service.

## External Services

| Service | Function |
| ------- | -------- |
| Redis | Worker retrieves object and file metadata to get identifiers that it will look up in Registry. It flags files being reingested and files needing to be saved, then saves the info back to Redis so the preservation uploader will know which files to copy to long-term storage, and what UUIDs to use as keys (S3 file names).
| Registry | Source of WorkItem record describing work to be done. The worker queries Registry to look for existing object and file records that would indicate that the current bag is a reingest.
| NSQ | Distributes WorkItem IDs to workers and tracks their status.

## Source Files

| Worker | Service | Files | Definition |
| ------ | ------- | ----- | ---------- |
| Reingest Manager | Ingest | [Task](https://github.com/APTrust/preservation-services/blob/master/ingest/reingest_manager.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/reingest_manager.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/reingest_manager/reingest_manager.go){target=_blank} | Checks if a bag is being reingested and if so, applies special processing. |
