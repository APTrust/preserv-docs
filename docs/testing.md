# Testing

Because workflow pipelines in preservation services are long and involve many steps, it is essential to maintain a full set of unit, integration, and end-to-end tests. Without these, it's impossible to detect subtle regressions caused by new features or changes to settings.

In general, Preserv functions should be broken down into the simplest possible units and should have no side effects. As far as possible, each function should be testable, and unit tests should test for both successful outcomes and for potential failures.

These goals are not always possible, especially for functions that interact with outside resources, but we should take them as guidelines in our coding practice.

## TL;DR

To run the entire test suite:

```
./scipts/test.rb integration && ./scripts/test.rb e2e
```

Nerds and masochists, read on.

## The Test Script

The `test.rb` script in the scripts folder manages unit, integration and end-to-end tests. In addition to running test suites, it orchestrates the start-up and shut down of external services such as Minio, NSQ, Redis and Registry. That is, it runs a full APTrust environment on your local machine, with a local Minio server standing in for S3 and Glacier.

For help, simply run `./scripts/test.rb` with no arguments.

## Unit Tests

To run unit tests, call this from the Preserv's top-level directory:

`./scripts/test.rb units`

The script will do the following:

* Delete and recreate folders in $HOME/tmp
* Start a local Redis server using the binary in the project's bin directory
* Run all of the unit tests & print results
* Shut down Redis and Minio

Files in $HOME/tmp remain after the tests complete. You can inspect logs in $HOME/tmp/logs and S3 items in $HOME/tmp/minio.

## Integration Tests

`./scripts/test.rb integration` will run the integration test suite. This does everything described in the unit tests above, plus the following:

* Start an instance of NSQ from the local bin directory
* Start an instance of Registry
* Empty the test database, `apt_registry_integration`
* Load all fixtures into `apt_registry_integration`
* Starts all Preserv services, including the ingest_bucket_reader, apt_queue, and apt_queue_fixity, which are responsible for pushing work items into NSQ
* Run integration tests & print results
* Shut down the Registry

After integration tests run, you can inspect artifacts left in the ~/tmp directory and in the database. Note that the integration test suite also runs the unit test suite.

## End to End Tests

`./scripts/test.rb e2e` runs end-to-end tests.

Preserv's end-to-end tests are the most likely to catch regressions. These tests ingest, delete, and restore a number of files and objects, and run fixity checks. They also test business logic such as two-factor deletion and alert generation.

The tests exercise all of Preserv's major functions and then test post-conditions in S3, Redis, NSQ, and Registry. For example, if an ingest succeeded, the expected post-conditions include:

* all files present in the right S3 (local Minio) buckets
* all interim files deleted from the S3 (local Minio) staging bucket
* interim data deleted from Redis
* object present in Registry
* all files present in Registry
* all expected checksums, storage records and premis events in Registry
* work item complete in Registry
* nsq item marked complete

The end-to-end tests run through expected success and expected failure scenarios. Like other test suites, they leave artifacts in ~/tmp and in the Registry's `apt_registry_integration` database. To examine the state of the Registry after end-to-end tests, change into the Registry's root directory and run `APT_ENV=integration ./registry serve`.

## Putting It All Together

The following command runs all the tests:

`./scipts/test.rb integration && ./scripts/test.rb e2e`

Note the the integration test suite implicitly runs the unit tests. Also note that the `&&` will prevent the end-to-end tests from running if the integration tests fail.

## Interactive Tests with DART 3

`./scripts/test.rb interactive` will not run any automated tests, but will spin up a whole APTrust environment on which you can run manual tests. The environment includes Minio, NSQ, Redis, and Registry.

This option is useful to manually test ingests, restorations and deletions. It's also handy in cases where you want to test a fix for a bag that failed ingest on a live system due to an error. Did your fix work? Try this:

1. Download the failed bag from the depositor's receiving bucket.
2. Run `./scripts/test.rb interactive`
3. Copy the bag into ~/tmp/minio/aptrust.receiving.test.test.edu

The bucket reader will find the bag in a few seconds and queue it for ingest. You can watch it go through at http://localhost:8080. The logins should be pre-populated.

You can tail the logs in ~/tmp/logs to see what the workers are doing. And you can look directly into the Redis interim processing data. See [Querying Redis](/components/redis#querying-redis) for more info.

To create some new bags and push them through the system, clone the dart-runner repo from https://github.com/APTrust/dart-runner and then at the command prompt:

1. Change into the top-level directory of dart-runner.
2. Start DART 3 with the command `go run dart/main.go`.

Now go to `http://localhost:8444` and you'll see the DART 3 UI. From here, you can create a new bag and upload it to the local Minio service for ingest.

First, create a new Storage Service that points to the local Minio instance, so you can upload the bag to a bucket where preservation services will see it:

1. Choose **Settings > Storage Services** from the top menu of DART 3.
1. Click the **New** button.
1. Apply the settings in the table below.

| Name | Value |
| ---- | ----- |
| Name | Local Minio |
| Description | test.edu receiving bucket on local Minio instance |
| Protocol | s3 |
| Host | localhost |
| Port | 9899 |
| Bucket | aptrust.receiving.test.test.edu |
| Allows Upload | Yes |
| Allows Download | Yes |
| Access Key ID | minioadmin |
| Secret Access Key | minioadmin |

Save the new Storage Service, then click on it in the list of Storage Services. You sould see a **Test Connection** button in the top right corner. Click that, and you should see a message that says the connection succeeded.

Now create a job to upload a new bag to the locally-running preservation services:

1. Choose `Jobs > New` from DART 3's top menu.
1. Drag some files from the left pane to the drop zone on the right to include them in the bag.
1. On the packaging screen, choose **BagIt** and the format, and **APTrust** as the BagIt profile. Give your bag a name and click next.
1. Fill out the required fields on the metadata page and click **Next**.
1. On the Upload Targets page, check the box next to **Local Minio - test.edu** and then click the **Next** button.
1. Click the green **Run Job** button.

You should see DART create, validate, and upload the bag. Now go to Registry at `http://localhost:8080` and check the Work Items page. You should see that preservation services is ingesting the bag you just uploaded.

If you want to look into the locally running Minio instance, go to `http://localhost:9899` if preservation services is running Minio on bare metal or `http://localhost:9001` if it's running in Docker. You can log in with username `minioadmin` and password `minioadmin`. You can browse the buckets from there.

You can kill the running DART 3 instance with Command-C in the terminal that lauched DART 3. You can kill the locally running preservation services/registry/minio/redis cluster with a Control-C in the terminal that launched it.

**Note**: Each time you run preservation services' `./scripts/test.rb interactive` command, it wipes out and resets your local Registry data.

## Local bin Directory

Inside the preservation-services repo is a bin directory containing OS-specific binaries of Redis, NSQ and Minio. The `test.rb` script detects which operating system you're currently running and selects the right binaries from either `bin/linux` or `bin/osx`.
