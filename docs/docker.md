# Docker

All Preserv components run in Docker containers in Amazon's Fargate service. All except the three cron jobs, `ingest_bucket_reader`, `apt_queue` and `apt_queue_fixity`, scale automatically based on ECS triggers. The cron jobs don't need to scale because their workload is so light.

There are 17 containers in all. Each container includes a single executable. The [Makefile](https://github.com/APTrust/preservation-services/blob/master/Makefile){target=_blank} generates the containers with the following commands:

```
make release
make update-template
```

The first command, `make release` builds the containers and pushes them to Docker hub. The second cmmand, `make update-template` updates the CloudFormation template to point to the newly built containers. (Each container is tagged with a Git commit ID. Newly built containers are tagged with the most recent commit ID. `make update-template` tells CloudFormation to pull the containers with the latest commit ID.)

## CI Testing and Docker Builds

Travis CI will run the Preserv unit tests, and if they pass, will run `make release`, pushing new containers to Docker Hub. Travis does not run `make update-template`. You'll have to do that on your own and then deploy if you want the new containers to run on AWS.

!!! Warning

    Travis is not currently set up to run the full Preserv test suite, so
    just because Travis tests pass doesn't mean Preserv is safe to deploy.

    We intend to fix this deficiency when we move from Travis to GitLab.

    Until then, read the next section, Testing Before Deployment!

## Testing Before Deployment

Travis runs only the unit tests. You should not deploy Preserv to any environment without running the integration and end-to-end tests first.

Currently, you can only run those locally. See [Putting it All Together](/testing/#putting-it-all-together) for info on how run the full test suite, and note that the suite takes several minutes to run.


## Command-line Deployment

TO BE FILLED IN LATER

## Deployment Through the AWS Console

After you've built new containers, follow these steps to deploy them manually through the AWS console.

1. Login to the AWS using your administrator account.
1. Proceed to the [CloudFormation Console](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringStatus=active&filteringText=&viewNested=true&hideStacks=false){target=_blank}
1. Select the stack `preserv-services-<env>` link to open stack details and options. (Where `<env>` is one of `prod`, `demo`, or `staging`.)
1. Select update.
1. Select replace current template
1. Select upload template file.
1. Upload the updated ‘cfn-preserv-master.yml’
1. Select next.
1. Select next again. ( You do not need to change any settings.)
1. At the final pane for deployment you will need to select that you know you are creating new resources.
1. Submit.

You can then monitor changes under the events tab. Failures will roll back automatically. Give about 10 minutes before worrying it has failed.

## List of Containers

The Preserv suite includes 17 containers. The workers inside these containers are compiled by the [Makefile](https://github.com/APTrust/preservation-services/blob/master/Makefile){target=_blank} from sources in Preserv's [apps directory](https://github.com/APTrust/preservation-services/tree/master/apps){target=_blank}

| Name | Executable | Service | Description |
| ---- | ---------- | ------- | ----------- |
| [Bucket Reader](/workers/ingest/bucket-reader) | ingest_ bucket_ reader | Ingest | A cron job that scans for new items in receiving buckets. Creates an ingest work item in Registry and pushes the work item ID into NSQ.
| [Metadata Gatherer](/workers/ingest/pre-fetch) | apt_ pre_fetch | Ingest | Streams a bag from a receiving bucket through a number of functions to calculate checksums and parse tag files and manifests. Saves tag files and manifests to S3 staging bucket. Saves all other metadata to Redis.
| [Bag Validator](/workers/ingest/validator) | ingest_ validator | Ingest | Analyzes the metdata gathered by apt_pre_fetch to ensure bag is valid. If bag is invalid, processing stops here. | ingest02_ bag_ validation
| [Reingest Manager](/workers/ingest/reingest-manager) | reingest_ manager | Ingest | Checks to see if the bag has ever been ingested before. If so, checks to see which files are new or updated.
| [Staging Uploader](/workers/ingest/staging-uploader) | ingest_ staging_ uploader | Ingest | Unpacks the tarred bag from the receiving bucket and stores its individual files in a temporary staging bucket, where other workers can access them.
| [Format Identifier](/workers/ingest/format-identifier) | ingest_ format_ identifier | Ingest | Streams files from the staging bucket through the Siegfried format identifier, which matches byte streams against a Pronom registry.
| [Preservation Uploader](/workers/ingest/preservation-uploader) | ingest_ preservation_ uploader | Ingest | Uploads files to long-term preservation buckets in S3, Glacier, and/or Wasabi.
| [Preservation Verifier](/workers/ingest/preservation-verifier) | ingest_ preservation_ verifier | Ingest | Verifies that the files copied into long-term preservation actually arrived intact and are accessible.
| [Ingest Recorder](/workers/ingest/recorder) | ingest_ recorder | Ingest | Records details of an ingest in the Registry.
| [Cleanup](/workers/ingest/cleanup) | ingest_ cleanup | Ingest | Cleans up temporary resources no longer required after ingest. These include files in the staging bucket, metadata records in Redis, and the tarred bag in the receiving bucket.
| [Queue Fixity](/workers/fixity/) | apt_ queue_ fixity | Fixity Check | Cron job that queues Generic Files for scheduled fixity checks.
| [Fixity Checker](/workers/fixity/) | apt_ fixity | Fixity Check | Worker that permforms scheduled fixity checks.
| [Glacier Restorer](/workers/restoration/#glacier-restoration) | glacier_ restorer | Restoration | Moves files from Glacier and Glacier Deep Archive into S3 so they can be restored.
| [File Restorer](/workers/restoration/#file-restoration) | file_restorer | Restoration | Restores individual files to depositor restoration buckets.
| [Bag Restorer](/workers/restoration/#object-restoration) | bag_ restorer | Restoration | Restores entire bags (intellectual objects) to depositor restoration buckets.
| [Deletion Worker](/workers/deletion/) | apt_delete | Deletion | Permanently deletes files and objects from preservation storage.
| APT Queue | apt_queue | Restoration and Deletion | Queues deletion and restoration requests created by Registry users. Those requests should be queued automatically by Registry itself. If they're not, `apt_queue` will find them. This cron job is a vestige from the old, unreliable Pharos system, which did occasionally fail at queueing requests. It may no longer be needed, but we'll keep it around as a failsafe.
