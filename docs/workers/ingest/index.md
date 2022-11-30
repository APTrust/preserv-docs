# Ingest Workers

The ingest process consists of 10 workers: one is cron job and the rest are service workers whose tasks are orchestrated by NSQ.

In the table below, the Name is the friendly name of the service or worker, while the Executable is the name of the compiled binary. "Reads From" describes the name of the NSQ topic from which the worker gathers its tasks. "Pushes To" is the name of the NSQ topic to which a worker pushes its task upon successful completion.

Except for the Bucket Reader, all workers use Redis to keep track of interim processing data. They read from Redis to see what past workers have done, and they write to Redis to tell future workers what they have done. For more on what goes into Redis, see the [Redis documentation](/components/redis/)

Click the name of any item in the table for more details.

| Order | Name | Executable | Description | Reads From | Pushes To |
| ----- | ---- | ---------- | ----------- | ---------- | --------- |
| 0 | [Bucket Reader](bucket-reader.md) | ingest_ bucket_ reader | A cron job that scans for new items in receiving buckets. Creates an ingest work item in Registry and pushes the work item ID into NSQ. | None | ingest01_ prefetch |
| 1 | [Metadata Gatherer](pre-fetch.md) | apt_ pre_fetch | Streams a bag from a receiving bucket through a number of functions to calculate checksums and parse tag files and manifests. Saves tag files and manifests to S3 staging bucket. Saves all other metadata to Redis. | ingest01_ prefetch | ingest02_ bag_ validation |
| 2 | [Bag Validator](validator.md) | ingest_ validator | Analyzes the metdata gathered by apt_pre_fetch to ensure bag is valid. If bag is invalid, processing stops here. | ingest02_ bag_ validation | ingest03_ reingest_ check |
| 3 | [Reingest Manager](reingest-manager.md) | reingest_ manager | Checks to see if the bag has ever been ingested before. If so, checks to see which files are new or updated. | ingest03_ reingest_ check | ingest04_ staging |
| 4 | [Staging Uploader](staging-uploader.md) | ingest_ staging_ uploader | Unpacks the tarred bag from the receiving bucket and stores its individual files in a temporary staging bucket, where other workers can access them. | ingest04_ staging | ingest05_ format_ identification |
| 5 | [Format Identifier](format-identifier.md) | ingest_ format_ identifier | Streams files from the staging bucket through the Siegfried format identifier, which matches byte streams against a Pronom registry. | ingest05_ format_ identification | ingest06_ storage |
| 6 | [Preservation Uploader](preservation-uploader.md) | ingest_ preservation_ uploader | Uploads files to long-term preservation buckets in S3, Glacier, and/or Wasabi. | ingest06_ storage | ingest07_ storage_ validation |
| 7 | [Preservation Verifier](preservation-verifier.md) | ingest_ preservation_ verifier | Verifies that the files copied into long-term preservation actually arrived intact and are accessible. | ingest07_ storage_ validation | ingest08_ record |
| 8 | [Ingest Recorder](recorder.md) | ingest_ recorder | Records details of an ingest in the Registry. | ingest08_ record | ingest09_ cleanup |
| 9 | [Cleanup](cleanup.md) | ingest_ cleanup | Cleans up temporary resources no longer required after ingest. These include files in the staging bucket, metadata records in Redis, and the tarred bag in the receiving bucket. | ingest09_ cleanup | None |
