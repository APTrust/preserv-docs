# Restoration

When a depositor requests file or object restoration by clicking the Restore button in the Registry, the Registry creates a restoration WorkItem and queues the WorkItem ID in one of the following NSQ topics:

* restore_file, if the user wants to restore a single file
* restore_object, if the user wants to restore an entire intellectual object (bag)
* restore_glacier, if the object of file being restored must first be retrieved from Glacier

Each of these topics has a different worker. The `glacier_restorer` subscribes to the restore_glacier topic. `file_restorer` subscribes to the restore_file topic, and `bag_restorer` subscribes to the restore_object topic.

## Glacier Restoration

If a file or object is stored in Glacier only, it must first be moved from the Glacier storage tier to the S3 storage tier before we can access it.

The `glacier_restorer` worker sends requests to AWS to move items from Glacier and Glacier Deep Archive temporarily into the S3 tier. That may take up to five hours for Glacier files and up to 12 hours for files in Glacier Deep Archive.

After making the initial request, `glacier_restorer` polls AWS's Glacier service every few hours to see if the files have been moved. When the files have reached the S3 tier, `glacier_restorer` pushes the WorkItem ID of the restoration request into either NSQ's `restore_file` or `restore_object` topic for completion. From there, the restoration proceeds like a normal S3 restoration.

## File Restoration

The `file_restorer` restores individual files from S3 and Wasabi into depositor's restoration buckets. Individually restored files will appear in the restoration bucket using the file identifier as the key.

For example, the Generic File with identifier `virginia.edu/photos/data/superbowl.jpg` will be restored to `aptrust.restore.virginia.edu/virginia.edu/photos/data/superbowl.jpg`

The file_restorer calculates checksums as it streams the file from the preservation bucket to the restoration bucket. If the calculated checksums don't match what's in the Registry, the restoration will be marked as Failed.

Once the file is restored, the WorkItem is marked complete, and the URL of the restored file appears in the WorkItem Note.

## Object Restoration

Object restoration involves restoring all of the files that make up an intellectual object, packaging them in BagIt format, and writing the entire tar file into the depositor's restoration bucket. The key of the restored bag will be the intellectual object identifier, minus the insitutional prefix, plus a ".tar" extension.

For example, the object `virginia.edu/photos` will be restored to `aptrust.restore.virginia.edu/photos.tar`. It will be a single file at the top level of the bucket, not nested in a subfolder, like a restored file.

The `bag_restorer` does the following when restoring an object:

* streams all of the objects through a checsum calculator into a tar file in the depositor's restoration bucket
* builds manifests as the files stream through
* writes tag files and manifests into the tar stream
* validates the bag
* marks the WorkItem complete, with the URL of the restored bag in the WorkItem note.

Validation occurs during the bagging process, so we don't have to re-read the tar file from the restoration bucket. Validation will fail if any files that Registry says are part of the bag are missing from storage or have invalid checksums.

Note that we keep the original bag-info.txt and aptrust-info.txt files that we received during the last ingest of this bag, and we restore them to the tar file.

!!! Note

    A restored bag will not exactly match the originally submitted bag,
    because the bag_restorer may add files to the payload directory in
    any order it likes, rather than in the order they were originally
    added.

    In addition, depositors sometimes delete files from an object
    between initial ingest and restoration. Deleted files will not be
    restored, so the restored bag may have fewer files than the original
    ingest.

## Resources

The `glacier_restorer` uses virtually no CPU, memory or network I/O. It simply issues periodic requests to AWS with small request and response sizes.

The `file_restorer` uses minimal resources when restoring smaller files, and may use substantial resources when restoring very large files. CPU usage goes up as it calculated checksums on large files. Memory usage can be somewhat high for large files (> 100 GB) and network usage is proportional to file size.

The `bag_restorer` can use considerable memory, CPU and bandwidth when restoring large bags or bags with high file counts.

## External Services

| Service | Function |
| ------- | -------- |
| S3 Preservation Buckets | Long-term storage area from which files are restored.
| Glacier Preservation Buckets | Long-term storage area from which files are restored.
| Glacier Deep Archive Buckets | Long-term storage area from which files are restored.
| Wasabi Buckets | Long-term storage area from which files are restored.
| S3 Restoration Buckets | Depositor buckets to which files and bags are restored.
| Registry | Source of WorkItem record describing work to be done.
| NSQ | Distributes WorkItem IDs to workers and tracks their status.
