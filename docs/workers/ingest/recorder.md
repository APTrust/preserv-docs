# The Recorder

The recorder records all metadata from the ingest in the Registry. It gets the metadata from Redis, where information about the object and files has been built up by the previous workers as they processed the ingest.

For example, the metadata gatherer parsed the tag files and calculated the checksums, the reingest manager reassigned UUIDs where necessary, the format identifier set file mime types, and the preservation uploader created storage records describing where the files now live.

The recorder creates or updates an intellectual object record in the Registry with the following info:

* Object identifier
* Owning institution
* Storage option
* Internal identifier and description
* Bag Group Identifier
* BagIt Profile ID
* Various other metadata parsed from bag-info.txt and aptrust-info.txt

The recorder then creates object-level Premis events, including events for ingestion, identifier assignment, and access assignment.

Once the object record exists, the recorder creates Generic File records for each file in the bag. It also records Premis Events, Checksums and Storage Records for each Generic File. The recorder sends Generic Files (with related records) in batches of 100, and it knows how to separate new files (POST/insert) from reingests (PUT/update).

## Resources

The recorder creates quite a bit of network chatter between itself, Redis and Registry. It uses a moderate amount of CPU and memory.

!!! Note

    Recording bags that have thousands, or even hundreds of thousands of files
    can be taxing on the Registry and the underlying Postgres database. It's
    normal for Registry and database performance to degrade during large
    recording operations.

## External Services

* Redis
* Registry
* NSQ
