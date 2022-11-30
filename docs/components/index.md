# Components

Preservation Services (aka Preserv) consists of the following components. Click on any component name to find out more about it.

| Component | Description |
| --------- | ----------- |
| [Redis / Elasticache](redis.md) | A shared data store where workers keep metadata about the ingest process. |
| [NSQ](nsq.md) | A queue service that distributes tasks to ingest, restoration, deltion, and fixity workers. |
| [Registry](registry.md) | A REST API providing access to the authoritative, permanent data store describing all items in preservation storage and all events related to those items. The Registry also contains authoritative WorkItem records that track the status of all ingest, restoration and deletion tasks. |
| [S3](s3.md) | The storage service used throughout all phases of ingest, restoration, preservation, fixity checking, and deletion. The term "S3" includes Glacier and Wasabi storage. |
