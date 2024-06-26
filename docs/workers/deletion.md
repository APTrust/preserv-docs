# Deletion

Depositors request deletions through the Registry UI. Once a deletion has been requested and approved by an institutional admin, the Registry creates a deletion WorkItem and pushes the WorkItem ID into NSQ's `delete_item` topic.

The `apt_delete` worker reads from this topic and deletes files or entire intellectual objects, as requested.

For single file deletions, the worker deletes all copies of the file from preservation storage, creates a deletion Premis event for the file in Registry, and marks changes the state of the Generic File object in Registry from "A" (active) to "D" (deleted).

For intellectual object deletions, the worker does the above for all of the object's files, then created a deletion Premis event for the object and changes its state from "A" to "D."

Finally, it marks the WorkItem complete.

For items using the `Standard` storage option, `apt_delete` expunges both the S3 copy and the Glacier copy of every file. All other storage options keep files in a single bucket or vault, so `apt_delete` has to expunge from only a single location.

!!! Note
    Also see the section on [Bulk Deletions](../overview#bulk_deletions)
    on the overview page. APTrust has internal documentation on how to
    create bulk deletion requests and the safeguards surrounding the
    process. See the internal docs for more information.

## Resources

`apt_delete` uses little CPU, memory, and network bandwidth.

## External Services

| Service | Function |
| ------- | -------- |
| S3 Preservation Buckets | Long-term storage area from which files are deleted.
| Glacier Preservation Buckets | Long-term storage area from which files are deleted.
| Glacier Deep Archive Buckets | Long-term storage area from which files are deleted.
| Wasabi Buckets | Long-term storage area from which files are deleted.
| Registry | Source of WorkItem record describing work to be done. Deletion workers update files and objects, and create deletion Premis events here.
| NSQ | Distributes WorkItem IDs to workers and tracks their status.

## Source Files

| Worker | Service | Files | Definition |
| ------ | ------- | ----- | ---------- |
| Deletion Manager | Deletion | [Task](https://github.com/APTrust/preservation-services/blob/master/deletion/manager.go){target=_blank} <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/deleter.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/apt_delete/apt_delete.go){target=_blank} | Deletes files and objects from preservation storage. |
| APT Queue | Deletion and Restoration | No Task File <br/> [Worker](https://github.com/APTrust/preservation-services/blob/master/workers/apt_queue.go){target=_blank} <br/> [App](https://github.com/APTrust/preservation-services/blob/master/apps/apt_queue/apt_queue.go){target=_blank} | This cron job periodically scans Registry for restoration and deletion requests that have not been queued in NSQ. |
