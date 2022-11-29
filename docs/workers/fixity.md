# Fixity

Preserv checks fixity on all files in S3 and Wasabi storage every 90 days. We do not run fixity checks on files in Glacier or Glacier Deep archives.

apt_queue_fixity runs as a cron job in its own container, querying Registry every half hour for files that have not had a fixity check in 90 days. It typically gets a batch of 2,500 files at a time, then pushes all of the Generic File IDs into NSQ's `fixity_check` topic.

From there, the `apt_fixity` worker picks them up, streaming each file through the sha256 checksum algorithm and creating a Premis event to record the result in Registry.

!!! Note

    Although we calculate multiple checksums on ingest, we calculate
    only sha256 digests during fixity checks.

    Also note that we do not create WorkItems for fixity checks because
    they would add no value to this one-step process, and creating them
    would add over 40 million unnecessary rows to the WorkItems table
    each year.

## Avoiding Duplicate Work

When a fixity batch contains a number of very large files (over 100 GB), the checker may not complete all 2,500 checks before apt_queue_fixity runs again. In that case, apt_queue_fixity will add duplicate files to NSQ's fixity_check topic. For example, the last 1,000 files that the checker did not complete on the prior run are added again as the first 1,000 files to be checked on the next run.

The fixity checker includes code to dedupe these files, so it will only check each file once. If the checker didn't do this, and it got behind on several consecutive runs, it could lead to a situation where the checker keeps checking the same set of files over and over again, never advancing to checker newer files that actually need to be looked at.

## Settings

You can change how often the apt_queue_fixity runs by adjusting the QUEUE_FIXITY_INTERVAL setting in AWS Parameter Store. You can adjust how many items are queued on each run by updating MAX_FIXITY_ITEMS_PER_RUN.

You can calso adjust MAX_DAYS_SINCE_LAST_FIXITY on the staging and demo systems to make fixity checks run more or less frequently. In production, that setting should always be set to 90 (days), because that's the interval our depositor agreement specifies.

## Resources

Fixity checks use substantial bandwidth and CPU. Checking large files may also use substantial memory.

## External Services

This worker talks to:

* S3 Preservation Buckets
* Wasabi Preservation Buckets
* Registry
* NSQ
