# Deletion

Depositors request deletions through the Registry UI. Once a deletion has been requested and approved by an institutional admin, the Registry creates a deletion WorkItem and pushes the WorkItem ID into NSQ's `delete_item` topic.

The `apt_delete` worker reads from this topic and deletes files or entire intellectual objects, as requested.

For single file deletions, the worker deletes all copies of the file from preservation storage, creates a deletion Premis event for the file in Registry, and marks changes the state of the Generic File object in Registry from "A" (active) to "D" (deleted).

For intellectual object deletions, the worker does the above for all of the object's files, then created a deletion Premis event for the object and changes its state from "A" to "D."

Finally, it marks the WorkItem complete.

For items using the `Standard` storage option, `apt_delete` expunges both the S3 copy and the Glacier copy of every file. All other storage options keep files in a single bucket or vault, so `apt_delete` has to expunge from only a single location.

## Resources

`apt_delete` uses little CPU, memory, and network bandwidth.

## External Services

This worker talks to:

* S3 Preservation Buckets
* Glacier Preservation Buckets
* Glacier Deep Archive Buckets
* Wasabi Buckets
* Registry
* NSQ
