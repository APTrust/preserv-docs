# Overview

All Preserv workers run in Docker containers on Amazon's ECS service, which adds and removes containers as the current workload requires.

Preserv workers also share the following capabilities, inherited from the [Base Worker](https://github.com/APTrust/preservation-services/blob/master/workers/base.go){target="_blank"}:

* A Context object providing connections to NSQ, Redis, and the Registry.
* A set of common methods described in the ServiceWorker interface for handling workflow decisions and storing and fetch housekeeping data.
* Channels for processing tasks and errors.
* A structure for gracefully handling unexpected shutdowns sent via SIGINT and SIGTERM.

Note that because some containers run on AWS spot instances, we do get unexpected SIGINT and SIGTERM.

## Framework and Configuration

While each worker has a different set of responsibilities, each essentially follows the same pattern in its workflow:

1. Get a message from NSQ.
2. Get a WorkItem record from Registry and determine whether to work on the item or skip it. (The fixity checker get a Generic File record instead of a WorkItem.)
3. If the item needs work, update the WorkItem in Registry to say it's being worked on.
4. Tell NSQ it's being worked on.
5. Get data from Redis describing the current state of processing for this item. The Redis data was created by prior workers, or by a previous processing attempt by the same worker.
6. Do your task (validate the bag, copy its files to a staging bucket, etc.), updating interim data Redis as you go.
7. If the worker failed, push it into the error channel to log details about the failure. Otherwise...
8. Update the WorkItem in Registry to say the worker did its job and the item is ready for the next step.
9. Mark the NSQ message as complete.
10. Push the item into the next NSQ topic.

While the Base worker provides a mechanism for completing all these steps, the specific workers supply a policy describing to the Base:

* What needs to be done during processing.
* When items should be skipped.
* How errors should be handled.
* Which NSQ topics successful and failed items should go into.

Each worker's policy is implemented as a set of fields and functions. For example, `NextQueueTopic` is a simple field on each worker describing which NSQ topic a message should go into after the worker successfully completes its work.

Meanwhile, each worker supplies a structure containing a single `Run()` method to the Base worker. The Base worker walks through its usual framework steps described above, calls `Run()` when appropriate, and then looks as the ProcessingError object returns by `Run()` to determine what to do next.

# Non-Fatal Errors

If the error is nil, the Base worker follows steps 7-10 above, pushing the item along to the next worker.

If `Run()` returns an error marked as non-fatal, the worker requeues the task to be picked up again in a few minutes by any instance of the same worker type. (E.g. If the preservation uploader requeues an item, any other instance of preservation uploader can pick it up later.)

When the item is picked up again, the worker fetches the interim processing data from Redis and does only the work that has not yet been completed. For example, it's common for the preservation uploader to encounter network errors, especially when uploading large sets of files. If it fails, the next uploader can see that 95 of the 100 files have already been successfully uploaded, and it will upload only the remaining 5.

If processing fails with a non-fatal error too many times, the WorkItem is marked as **NeedsAdminReview** and processing stops. An APTrust admin can requeue the item in the Registry when appropriate.

The most common non-fatal errors are network connection errors and unavailable service errors (e.g. Registry, Redis, S3, Glacier, or Wasabi cannot respond to requests). It makes sense for the APTrust admin to wait to requeue these items until the network and external services are known to be in a healthy state.

Note that each worker's MAX_ATTEMPTS setting in Parameter Store determines how many non-fatal errors it will tolerate before giving up on an item.

# Fatal Errors

If `Run()` returns a fatal error, the worker does the following:

* Logs detailed info on the fatal error.
* Sets the following attributes on the WorkItem and saves it to Registry:
    * Retry = false (so other workers won't retry it)
    * NeedsAdminReview = true (so APTrust admins know to look into it)
    * Note = a brief summary of the fatal erro
* Tells NSQ the item failed.

The most common fatal error is a bag validation error. Someone sent us a bag in the wrong format, or a bag whose manifest doesn't agree with its payload. We cannot ingest these bags. This error is truly fatal, and no further work should be done.

Less common fatal errors include:

* Lack of permissions on some essential external resource, such as an S3 bucket. This should not happen after initial launch.
* A rare problem recording data in Registry. This happens when the ingest recorder is killed by a SIGTERM or SIGINT between the moment of issuing a POST request and being able to process the Registry's response. This does happen on occasion when spot instances die. The symptom is an "identifier already exists" error in a WorkItem note. When this occurs, an APTrust admin should delete the WorkItem's registry data and requeue the item back to the `Receive/Pending` stage. Thanks to the reingest manager, the system is smart enough to handle this case intelligently.
