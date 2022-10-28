# Redis

Preserv uses Redis (running as Elasticache in AWS) to hold interim processing data that is accessible to all ingest workers. The first worker in the ingest pipeline, `ingest_pre_fetch`, reads the contents of a tarred bag from an S3 receiving bucket and records metadata, including:

* General info about the bag, including its name, owning institution, size, number of payload files, and more. The object info also includes parsed versions of the bag's tag files.
* General info about each file in the tarred bag, including it's path, size, and checksums from the bag's manifests.
* A "work result" describing when the worker started its task, when it completed, and any errors it encountered along the way.

The `ingest_pre_fetch` worker dumps this data into Redis, in JSON format, so the next worker can access it.

Each successive worker adds information to the object, file, and work result records in Redis so subsequent workers know what to do with the incoming content. For example, the `reingest_manager` checks to see if we already have copies of a bag's files in preservation storage. If so, and if the checksums have not changed, it flags the files as not needing to be saved.

The later workers `ingest_preservation_uploader` and `ingest_preservation_verifier` know not to copy these files to preservation storage and not to bother verifying the copies. The `ingest_recorder` knows not to record new ingest events or checksums for these files.

We chose Redis as our interim processing cache for a number of reasons:

* The workers often perform huge volumes of reads and writes that would bog down a relational database.
* The data naturally fits into a key-value data store. We don't want or need to decompose the JSON into searchable tables or MongoDB-type documents. We always want the whole blob.
* Interim processing data is ephemeral. We may retain it for a few seconds in the case of small bags, or 2-3 days in the case of larger bags. In most cases, we keep it for 1-10 minutes. Redis' disk-backed "aof" file, which allows in-memory data to persist through reboots, is enough of a durability guarantee for our needs.

By the time a bag reaches the `ingest_recorder` worker, the JSON data in Redis contains all of the information we will need to record in the Registry, including:

* The object record, with its title, description, owning institution and other metadata.
* All of the file records, and there can be tens of thousands of these.
* All checksums, premis events, and storage records for all files.

!!! Note

    On live systems, we configure Redis to write to an .aof file
    (append-only file) so that data persists through system reboots.
    On dev and test machines, using an .aof file is optional.

## Structure of Redis Data

The key for each WorkItem in Redis is the WorkItem ID. The value is a hash of subkeys. The value for each subkey is a blob of JSON data. Data for an ingest with WorkItem ID 5678 will look like this:

```
"5678"
   "object:example.edu/BagOfPhotos":            [JSON ~1-3 kb]
   "workresult:ingest01_prefetch":              [JSON ~200-500 bytes]
   "workresult:ingest02_bag_validation":        [JSON ~200-500 bytes]
   "workresult:ingest03_reingest_check":        [JSON ~200-500 bytes ]
   "workresult:ingest04_staging":               [JSON ~200-500 bytes ]
   "workresult:ingest05_format_identification": [JSON ~200-500 bytes ]
   "workresult:ingest06_storage":               [JSON ~200-500 bytes ]
   "workresult:ingest07_storage_validation":    [JSON ~200-500 bytes ]
   "workresult:ingest08_record":                [JSON ~200-500 bytes ]
   "workresult:ingest09_cleanup":               [JSON ~200-500 bytes ]
   "file:test.edu/BagOfPhotos/file001"          [JSON ~1-3 kb]
   "file:test.edu/BagOfPhotos/file002"          [JSON ~1-3 kb]
                   ...
              Lots more files
                   ...
   "file:test.edu/BagOfPhotos/file999"          [JSON ~1-3 kb]
```

## Redis Memory Usage

As you can see from the outline above, there's a direct linear relation between the number of files in a bag and the amount of interim processing data in Redis. In the past, we've ingested bags with over 300,000 files. Ingesting too many files at once can cause Redis to run out of memory. Consider the impact of Redis keep 300,000 3kb JSON blobs in memory at once.

Until we update Preserv's code to work with Redis clusters, we're dealing with this issue by running an Elasticache instance with more RAM than we'll typically need. When Preserv works with Redis running in cluster mode, we'll be able to take advantage of AWS auto-scaling for Elasticache.

Unfortunately, because of the way our workers connect to and query Redis, this will not be a trivial change.

## Querying Redis

You can query Redis directly using the redis-cli on your local machine. Simply run `redis-cli`, and the CLI will connect to your local Redis instance running on the default port of 6379.

To query Redis/Elasticache in one of the live environments, you'll have to log into a bastion host and run the redis-cli from there. You'll need to look in Parameter Store for the proper Redis/Elasticache endpoint for your current environment. You can then connect with:

`redis-cli -h <host name or ip> -p 6379`

### Getting a List of Keys / WorkItems

Redis records use WorkItem IDs as the main key, and then various subkeys for records related to that WorkItem ID. To get a list of all WorkItem IDs in Redis, run

`keys "*"`

You should see output like this:

```
 1) "24719"
 2) "24553"
 3) "24627"
 4) "24516"
 5) "24514"
 6) "24796"
 7) "24450"
 8) "24487"
 9) "24764"
10) "24564"
11) "24557"
12) "24657"
```

That shows Redis has records for twelve WorkItems with IDs ranging from 24719 to 24657. Note that because keys in Redis are strings, the IDs are strings, not numbers. The records stored under each WorkItem ID are hashes, each with its own key and value.

!!! Note

    Redis documentation warns against running `keys "*"` in production,
    as it may return millions of results. Because queue sizes are limited
    in our systems, and because we delete Redis data as soon as ingests
    complete, we will generally have fewer than 200 keys at any given
    time. This means it's generally safe to run `keys "*"` in production.
    However, see the note on listing records for a WorkItem below.

### Listing Records for a Key / WorkItem

To see what info Redis has about WorkItem 24657, run this:

`hkeys "24657"`

You'll see output like this:

```
 1) "file:virginia.edu/LibraETD-0r9673755/aptrust-info.txt"
 2) "workresult:ingest02_bag_validation"
 3) "file:virginia.edu/LibraETD-0r9673755/tagmanifest-md5.txt"
 4) "file:virginia.edu/LibraETD-0r9673755/manifest-sha256.txt"
 5) "file:virginia.edu/LibraETD-0r9673755/data/Phylindia Gant MA 2016"
 6) "file:virginia.edu/LibraETD-0r9673755/tagmanifest-sha256.txt"
 7) "file:virginia.edu/LibraETD-0r9673755/data/work.json"
 8) "file:virginia.edu/LibraETD-0r9673755/bagit.txt"
 9) "file:virginia.edu/LibraETD-0r9673755/manifest-md5.txt"
10) "file:virginia.edu/LibraETD-0r9673755/data/embargo.json"
11) "workresult:ingest01_prefetch"
12) "file:virginia.edu/LibraETD-0r9673755/bag-info.txt"
13) "object:virginia.edu/LibraETD-0r9673755"
14) "file:virginia.edu/LibraETD-0r9673755/data/fileset-1.json"
15) "file:virginia.edu/LibraETD-0r9673755/data/Lewis_Gwendolyn_2015.pdf"

```

Note that the keys follow a pattern. Those beginning with `file:` contain information about a file being ingested. Those beginning with `workresult:` contain info about the result of a step of the ingest process. So `workresult:ingest01_prefetch` has info about the result of the pre-fetch step of ingest, while `workresult:ingest02_bag_validation` has info about the validation step.

Each ingest WorkItem has a single `object:` key, composed of the object prefix plus the object identifier. In the case above, it's `object:virginia.edu/LibraETD-0r9673755`. This key contains information about the object and is discussed below.

!!! Note

    Some objects contain hundreds of thousands of files. Running `hkeys`
    on those will return more info than you want. Try querying the object
    record first and looking at the file count, as described under
    [Examining an Object Record](#examining-an-object-record) below.


### Examining a Work Result Record

All of the Redis data is stored in JSON format. Let's see what's in `workresult:ingest01_prefetch`. We can do that by running the hget command. Note that both the key (24657) and the field (workresult:ingest01_prefetch) must be quoted.

`hget "24657" "workresult:ingest01_prefetch"`

```json
{
	"attempt": 2,
	"operation": "ingest01_prefetch",
	"host": "ip-10-0-79-215.ec2.internal",
	"pid": 1,
	"started_at": "2022-10-20T17:37:16.742934015Z",
	"finished_at": "2022-10-20T17:37:17.665739918Z",
	"errors": null
}
```

This tells us that after two attempts, the pre-fetch worker succeeded (because there are no errors). We see when the worker started and finished, and the host field tells us which worker did the work.

### Examining a File Record

Now let's look at a file record.

`hget "24657" "file:virginia.edu/LibraETD-0r9673755/data/Lewis_Gwendolyn_2015.pdf"`

```json
{
	"checksums": [{
		"algorithm": "md5",
		"datetime": "2022-10-20T17:36:14.854845867Z",
		"digest": "813f03c186f0cf350e4a7b23eb1c757e",
		"source": "manifest"
	}],
	"copied_to_staging_at": "0001-01-01T00:00:00Z",
	"format_identified_at": "0001-01-01T00:00:00Z",
	"file_modified": "0001-01-01T00:00:00Z",
	"is_reingest": false,
	"needs_save": true,
	"object_identifier": "virginia.edu/LibraETD-0r9673755",
	"path_in_bag": "data/Lewis_Gwendolyn_2015.pdf",
	"registry_urls": [],
	"saved_to_registry_at": "0001-01-01T00:00:00Z",
	"size": 0,
	"storage_option": "Standard",
	"storage_records": [],
	"uuid": "bf9d5811-0ad6-43f1-8406-83e14735b4a1"
}
```

This tells us that the file arrived with one checksum in the bag manifest, an md5. It also tells us the UUID that will become the file name in the preservation bucket. This file is bound for Standard storage, but has not yet been copied to staging, run through format identification, or saved to the registry, as those timestamps are all empty.

This file record will expand as additional workers do their processing. The validation worker will add its own checksums to the checksum list (with source = "ingest" instead of source = "manifest" so we know that we calculated the checksums). The reingest worker will see if this file has ever been ingested before, and may change needs_save to false if the checksum matches the checksum of the copy we already have.

### Examining an Object Record

Runing this command to get the object record:

`hget "24657" "object:virginia.edu/LibraETD-0r9673755"`

We get the following results:

```json
{
	"copied_to_staging_at": "0001-01-01T00:00:00Z",
	"deleted_from_receiving_at": "0001-01-01T00:00:00Z",
	"etag": "ed17d88a33e8c095ab3674e08184b398",
	"file_count": 11,
	"has_fetch_txt": false,
	"institution": "virginia.edu",
	"institution_id": 34,
	"is_reingest": false,
	"manifests": ["md5", "sha256"],
	"parsable_tag_files": ["bagit.txt", "bag-info.txt", "aptrust-info.txt"],
	"recheck_registry_identifiers": false,
	"s3_bucket": "aptrust.receiving.test.virginia.edu",
	"s3_key": "LibraETD-0r9673755.tar",
	"saved_to_registry_at": "0001-01-01T00:00:00Z",
	"serialization": "application/tar",
	"should_delete_from_receiving": true,
	"size": 1576960,
	"storage_option": "Standard",
	"tag_files": ["bagit.txt", "bag-info.txt", "aptrust-info.txt"],
	"tag_manifests": ["md5", "sha256"],
	"tags": [{
		"tag_file": "bagit.txt",
		"tag_name": "BagIt-Version",
		"value": "0.97"
	}, {
		"tag_file": "bagit.txt",
		"tag_name": "Tag-File-Character-Encoding",
		"value": "UTF-8"
	}, {
		"tag_file": "bag-info.txt",
		"tag_name": "Bag-Count",
		"value": "1"
	}, {
		"tag_file": "bag-info.txt",
		"tag_name": "Bag-Group-Identifier",
		"value": "LibraETD"
	}, {
		"tag_file": "bag-info.txt",
		"tag_name": "Bag-Software-Agent",
		"value": "bagit.py v1.8.1 u003chttps://github.com/LibraryOfCongress/bagit-pythonu003e"
	}, {
		"tag_file": "bag-info.txt",
		"tag_name": "Bagging-Date",
		"value": "2022-06-23"
	}, {
		"tag_file": "bag-info.txt",
		"tag_name": "Internal-Sender-Description",
		"value": ""
		""
	}, {
		"tag_file": "bag-info.txt",
		"tag_name": "Internal-Sender-Identifier",
		"value": "0r9673755"
	}, {
		"tag_file": "bag-info.txt",
		"tag_name": "Payload-Oxum",
		"value": "1553270.4"
	}, {
		"tag_file": "bag-info.txt",
		"tag_name": "Source-Organization",
		"value": "virginia.edu"
	}, {
		"tag_file": "aptrust-info.txt",
		"tag_name": "Title",
		"value": ""
		Alluvial Fans as Potential Sites
		for Preservation of Bio - signatures on Mars ""
	}, {
		"tag_file": "aptrust-info.txt",
		"tag_name": "Description",
		"value": ""
		Water is an important central theme in astrobiology.In order
		for life as we know it to support itself on another planet,
		a of water is required.There are several s of water that have been discovered on Mars.At present,
		the majority of the s of water are unavailable,
		but there were periods in Marsxe2x80x99 geologic history that water was available
		for life.The next step in discovering life is looking
		for bio - signatures.Bio - signatures can be anything from isotopes to elements to DNA.A method to preserve the bio - signatures is necessary to study the bio - signatures in present day.Preserved bio - signatures keep information about extinct Mars habitability intact to study further when found.One naturally occurring way to preserve bio - signatures is clay hydrogels.Clay hydrogels are bio - polymers that form when clay minerals come in contact with and interact with water.The clay and water create a confining environment protecting DNA and RNA functions inside the polymer.Potential sites to look
		for clay hydrogels are alluvial fans,
		plentiful in the southern hemisphere of Mars.Alluvial fans are characterized as landforms that make a semi conical shape out of sediment deposits forming as water carries the sediment from a steep slope upstream to an unconfined drainage outlet downstream,
		often in flatter terrain.Due to their mineral composition,
		water origin,
		and location through - out Mars,
		alluvial fans are the perfect site to investigate in the search
		for preserved bio - signatures.
		""
	}, {
		"tag_file": "aptrust-info.txt",
		"tag_name": "Access",
		"value": "Consortia"
	}, {
		"tag_file": "aptrust-info.txt",
		"tag_name": "Storage",
		"value": "Standard"
	}, {
		"tag_file": "aptrust-info.txt",
		"tag_name": "Storage-Option",
		"value": "Standard"
	}]
}
```

There's quite a bit of information here. Most of it is used during bag validation, and then copied into the registry when ingest completes. Note that a WorkItem's object record is visible to APTrust admins on the Registry's WorkItem detail page.

The object's size and file count attributes can be useful in determining why an object moves slowly through the ingest process. Multi-terabyte objects take a long time to read and copy, while bags containing tens of thousands of files incur enormous overhead (checksumming, format identification, re-ingest lookups, copying to S3, etc. have to be performed on each file).
