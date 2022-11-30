# The Format Identifier

The format identifier uses [Siegfried](https://www.itforarchivists.com/siegfried){target=_blank} to identify file formats based on byte sequence signatures in the [PRONOM](http://www.nationalarchives.gov.uk/pronom/){target=_blank} registry.

It streams files one by one from the staging bucket through its identification algorithms and records the results in Redis. If you look at the [Redis file record](components/redis/#examining-a-file-record), you'll notice the following fields:

* file_format - this is the format as identified by the format identifier
* format_identified_by - this will be either "siegfried," indicating that Siegfried match the file to an entry in the PRONOM registry, or "ext map," indicating that the file could not be matched, so it was identified by its extension.
* format_match_type - describes how the match was determined. "signature" means it was matched by Sigfried comparing it to a PRONOM signature, "extension" means either Siegfried or a circuit breaker (see below) matched it by extension, or "container" means Siegfried matched it as a container type (e.g. tar, zip, jar, rar, or certain Microsoft Office file types that contain multiple internal file streams).

## Short Circuiting and Extension Matching

Siegfried may crash when attempting to identify certain container formats if the file is corrupt or internally inconsistent. This happens most often with proprietary Microsoft Office container formats as listed in the `CrashableFormats` map in the [format identifier source code](https://github.com/APTrust/preservation-services/blob/master/ingest/format_identifier.go){target=_blank}.

When the format identifier encounters a file with a crashable extension, it skips Siegfried's PRONOM-matching algorithms and matches on extension only. In this case, the `format_identified_by` attribute of the Redis file record will be set to `ext map`, and the `format_match_type` will be set to `extension`, indicating that Siegfried didn't even attempt to identify it.

In other cases, Siegfried may try and fail to do a byte signature match, then fall back to an extension match. In these cases, `format_identified_by` will be `siegfried` and `format_match_type` will be `extension`.

## Resources

Siegfried uses large amounts of network I/O and memory because it runs large files through a number of internal functions. Memory, in particular, is a problem in our low-resource Docker containers. To prevent out-of-memory exceptions, the format identifier containers process only one file at a time. This makes the format identifier a bottleneck in the ingest pipeline. This worker is also the most likely to scale to multiple instance even under light loads.

The format identifier still occasionally dies before completing its work. This may be due to out-of-memory exceptions, or it may be due to the worker running on spot instances that are killed by AWS because their owners want them back.

In either case, simply requeing the item in the Registry fixes the problem. Requeue to the Format Identification stage and the identifier will pick up where the dead worker left off.

## External Services

| Service | Function |
| ------- | -------- |
| S3 Staging Bucket | Worker streams files from staging through a format identification function to determine file format.
| Redis | Worker updates file records in Redis with file format and some metadata about how the file format was determined.
| Registry | Source of WorkItem record describing work to be done.
| NSQ | Distributes WorkItem IDs to workers and tracks their status.
