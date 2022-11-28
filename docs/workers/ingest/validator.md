# The Bag Validator

The pre-fetch worker (ingest_pre_fetch) streams bags from a receiving bucket through a series of analytical functions, parsing manifests and tag files, and calculating checksums. It leaves the results of its work in Redis. (See [Querying Redis](/components/redis/#querying-redis) for examples of what those results look like.)

After the pre-fetch worker completes, the bag validator, ingest_validator, examines all of the Redis data to ensure the following:

1. That the bag uses one of our supported BagIt profiles (either APTrust or BTR). The profile should be declared in the BagIt-Profile-Identifier tag in the bag-info.txt file. If it's missing, we assume the bag uses the APTrust profile, and we validate it accordingly.
2. That the bag conforms to the profile, meaning:
    1. It includes all required manifests and tag manifests.
    2. It does not include forbidden manifests or tag manifests.
    3. It does not include a fetch.txt file (we don't support this at all).
    4. All required tags files are present.
    5. All required tags are present and have legal values.
3. That all files mentioned in the manifests appear in the payload directory, and that all of their checksums match.
4. That the payload directory contains no extraneous files (i.e. nothing more than what is mentioned in the manifests).
5. That tag manifest checksums are correct.

Note that many profiles, including APTrust and BTR (Beyond the Repository) allow for the presence of additional files outside the payload directory that are not mentioned in payload manifests. APTrust treats all such files as custom tag files if they:
    1. are not in the payload (data) directory
    2. do not match any manifest file naming pattern (e.g. manifest-md5.txt, manifest-sha256.txt, etc.)
    3. do not match any tag manifest file naming pattern (e.g. tagmanifest-md5.txt, tagmanifest-sha256.txt, etc.)

## Invalid Bags Are Fatal Errors

If the validator determines a bag is invalid, it marks the WorkItem as failed and adds the specific validation errors to the WorkItem.Note field, which both the depositor and APTrust admins can see in the Registry.

Because we cannot ingest an invalid bag, no further work is done on the bag.

!!! Note

    The old Exchange ingest system, which we retired in November, 2022,
    used to automatically delete invalid bags from the receiving buckets.
    Preserv does not do that because its validator is not yet as battle-tested
    as Exchanges.

    In cases where it incorrectly marks bags as invalid (and there were
    several in the first week of production), we want to keep the bags
    available for analysis.

    We may have the validator automatically delete invalid bags after we've
    been in production long enough to trust it to handle odd edge cases.
    Until then, we rely on lifecycle policies in the receiving buckets to
    delete bags older than 60 days.

Bags that pass validation move into the `ingest03_reingest_check` topic in NSQ.

## Resources

The ingest_validator is very fast, using minimal CPU, memory, and network bandwidth. It generally completing its work in less than a second, because the metadata gatherer did most of the intensive work earlier.

The validator fetches manifests and tag manifests stored in the staging bucket. It verifies that the tag files conform to the BagIt profile, then compares the checksums in the manifests against the checksums stored in Redis.

For info on the structure of a bag's interim data in the staging bucket, see the [Staging Bucket](/components/s3/#staging-bucket) overview.

## External Services

This worker talks to:

* S3 staging bucket
* Redis
* Registry
* NSQ
