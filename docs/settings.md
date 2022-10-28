# Settings

## Settings Files

For developer and test setups, we store settings in simple .env files that each worker reads at startup. You'll notice files like .env.test in Preserv's source repo. You can tell workers which .env file to read by specifying APT_ENV on the command line.

For example, `APT_ENV=test apt_fixity` would run the apt_fixity worker using the configuration settings in .env.test. If you want to read settings from .env.local, run `APT_ENV=local apt_fixity`.

## Settings Injection Through AWS Parameter Store

For our staging, demo, and production environments, we ship an empty .env file inside the Docker container. Before Amazon's ECS service starts a new container instance, it runs a "sidecar" app that pulls variables from AWS parameter store into environment variables inside each container.

The code in  [config.go](https://github.com/APTrust/preservation-services/blob/master/models/common/config.go){target=_blank} uses [Viper's](https://github.com/spf13/viper){target=_blank} `AutomaticEnv` method to pull in environment variables. Any variables not defined in the .env file will be fetched from the environment, if present.

This allows us to use simple .env config files on developer machines and in CI tests, without exposing sensitive information in our public Git repos. For staging, demo, and production systems, all config settings are centralized in Parameter Store.

Note that variable names in Parameter Store follow the pattern `ENV/PRESERVE/VAR_NAME`, where ENV is the name of the environment (staging, demo, production) and VAR_NAME is the setting name. For example, the name of the variable containing the demo NSQ url is `/DEMO/PRESERV/NSQ_URL`.

## Definitions

The definitions below pertain to all Preserv workers. In addition to these, each worker has specific settings describing number of workers and buffer size. Those are described on worker-specific pages.

In general, settings ending in `WORKERS` describe how many go routines (concurrent processes) a worker should run. Settings ending in `BUFFER_SIZE` describe the desired size of internal queue buffers. On a macro level, the buffer size settings tell the work how many items to accept from NSQ in each batch.

If a worker has two `WORKERS` and a `BUFFER_SIZE` of 20, each go routine will accept up to 20 items at a time from NSQ. The Docker container then can be working on up to 40 NSQ items at a time.

| Variable Name | Definition |
|---------------|------------|
| BUCKET\_GLACIER\_DEEP\_OH | The name of the Glacier Deep preservation bucket in Ohio. This is for storage option Glacier-Deep-OH.  |
| BUCKET\_GLACIER\_DEEP\_OR | The name of the Glacier Deep preservation bucket in Oregon. This is for storage option Glacier-Deep-OR.  |
| BUCKET\_GLACIER\_DEEP\_VA | The name of the Glacier Deep preservation bucket in Virginia. This is for storage option Glacier-Deep-VA.  |
| BUCKET\_GLACIER\_OH | The name of the Glacier preservation bucket in Ohio. This is for storage option Glacier-OH.  |
| BUCKET\_GLACIER\_OR | The name of the Glacier preservation bucket in Oregon. This is for storage option Glacier-OR and Standard.  |
| BUCKET\_GLACIER\_VA | The name of the Glacier preservation bucket in Virginia. This is for storage option Glacier-VA.  |
| BUCKET\_STANDARD\_VA | The name of the S3 bucket for standard storage in Virginia. This is for storage option Standard. |
| BUCKET\_WASABI\_OR | The name of the Wasabi preservation bucket in Oregon. |
| BUCKET\_WASABI\_VA | The name of the Wasabi preservation bucket in Virginia.  |
| MAX\_DAYS\_SINCE\_LAST\_FIXITY | The interval at which the fixity checker should check files. In production and demo, this is set to 90, so that we run fixity checks every 90 days. In staging, you can set this down to one day if you want to force fixity checks to run. Typical staging setting is 14. |
| MAX\_FIXITY\_ITEMS\_PER\_RUN | The maximum number of files that the `queue_fixity` worker should queue on each run. The default value is 2500.  |
| NSQ\_LOOKUP | The hostname and port of the NSQ lookup daemon. This usually has a format like hostname:port or ip_addr:port. The lookup daemon tells NSQ clients (our workers) how to connect to any and all available NSQ instances. |
| NSQ\_URL | The hostname and port of our NSQ service. |
| PRESERV\_REGISTRY\_API\_KEY | The API key workers use to access the Registry. |
| PRESERV\_REGISTRY\_API\_USER | The API user email address that our workers use to access the Registry. This account must have the APTrust admin role, as it accesses the admin API. |
| PRESERV\_REGISTRY\_URL | The URL of the Registry. Workers read and write WorkItems and other data in this Registry.  |
| QUEUE\_FIXITY\_INTERVAL | This describes how often, in minutes, the queue fixity worker should check for files requiring fixity checks. We usually set this to 30.  |
| REDIS\_URL | The Redis (or Elasitiche) URL. Ingest workers connect to this service to store, retrieve and update interim processing data. On dev and CI machines, we use Redis. In AWS environments, we use Elasticache. |
| S3\_AWS\_HOST | The generic hostname for AWS S3: s3.amazonsws.com.  |
| S3\_AWS\_KEY | The Access Key ID used to access items in S3 buckets and in Glacier. This account should have full privileges in S3 and Glacier. |
| S3\_AWS\_SECRET | The AWS Secret Access Key used to interact with S3 and Glacier. |
| S3\_WASABI\_HOST\_OR | The hostname of Wasabi's Oregon S3 service. s3.us-west-1.wasabisys.com  |
| S3\_WASABI\_HOST\_VA | The hostname of Wasabi's Virginia S3 service. s3.us-east-1.wasabisys.com  |
| S3\_WASABI\_KEY | The Access Key ID used to access Wasabi buckets.  |
| S3\_WASABI\_SECRET | The Secret Access Key used to access Wasabi buckets.  |
| STAGING\_BUCKET | The name of the AWS S3 bucket into which the [staging uploader](/workers/ingest/staging-uploader.md) copies the files it unpacks from tarred bags in the receiving buckets. The [format identifier](workers/ingest/format-identifier.md) and other workers will access files in this bucket during later stages of ingest. |
