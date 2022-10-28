# Testing

Because workflow pipelines in preservation services are long and involve many steps, it is essential to maintain a full set of unit, integration, and end-to-end tests. Without these, it's impossible to detect subtle regressions caused by new features or changes to settings.

In general, Preserv functions should be broken down into the simplest possible units and should have no side effects. As far as possible, each function should be testable, and unit tests should test for both successful outcomes and for potential failures.

These goals are not always possible, especially for functions that interact with outside resources, but we should take them as guidelines in our coding practice.

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

## Interactive Tests

`./scripts/test.rb interactive` will not run any automated tests, but will spin up a whole APTrust environment on which you can run manual tests. The environment includes Minio, NSQ, Redis, and Registry.

This option is useful in cases where you want to test a fix for a bag that failed ingest on a live system due to an error. Did your fix work? Try this:

1. Download the failed bag from the depositor's receiving bucket.
2. Run `./scripts/test.rb interactive`
3. Copy the bag into ~/tmp/minio/aptrust.receiving.test.test.edu

The bucket reader will find the bag in a few seconds and queue it for ingest. You can watch it go through at http://localhost:8080. The logins should be pre-populated.

You can tail the logs in ~/tmp/logs to see what the workers are doing. And you can look directly into the Redis interim processing data. See [Querying Redis](/components/redis#querying-redis) for more info.

## Local bin Directory

Inside the preservation-services repo is a bin directory containing OS-specific binaries of Redis, NSQ and Minio. The `test.rb` script detects which operating system you're currently running and selects the right binaries from either `bin/linux` or `bin/osx`.
