# Code Structure

The preservation services [code repository](https://github.com/APTrust/preservation-services){target=_blank} includes the following directories and files:

| Name | Description |
| ---- | ----------- |
| .github | Contains config files for GitHub services like Dependabot. |
| apps | Contains Go source files with a `main()` function that can be compiled into executable binaries. There's one directory and one file for each app. |
| audit | Contains code to run a quick, lightweight audit of files in preservation storage. |
| bagit | Contains code used to build, parse, and validate BagIt packages. |
| bin | Contains Mac and Linux binaries for Minio, NSQ, and Redis. These are used in integration and end-to-end (e2e) tests. |
| cfn | Contains a template (`cfn-preserv-cluster.tmpl`) and a yaml file (`cfn-preserv-cluster.yml`) for deploying preservation services to AWS's Fargate/ECS service. |
| constants | Contains constants used throughout the codebase. |
| deletion | Contains code used by the deletion worker to permenantly remove files from preservation storage. |
| docker | Probably no longer used. Consider deleting this. |
| e2e | Contains end-to-end tests. See [End to End Tests](testing/#end-to-end-tests). |
| fixity | Contains code used by the fixity worker to run fixity checks. |
| ingest | Contains code used by ingest workers to ingest bags. |
| models | Contains a number of data models. See the following three entries for details. |
| models/common | Contains utility models used throughout preservation services. The most important of these are [Config](https://github.com/APTrust/preservation-services/blob/master/models/common/config.go){target=_blank}, which provides access to configuration settings and [Context](https://github.com/APTrust/preservation-services/blob/master/models/common/context.go){target=_blank}, which provides access to essential services such as logging, Registry, Redis, NSQ, S3, Glacier and Wasabi.  |
| models/registry | Contains models that match core Registry models (intellectual object, generic file, work item, checksum, etc.) These models include extra fields for metadata used during the ingest process that Registry itself doesn't care about. Data in these models is stored in Redis, so it can be shared among workers. |
| models/service | Contains models specific to preservation services' internal processing. Data in these models is stored in Redis, so it can be shared among workers. |
| network | Contains code to access external network services, including NSQ, Redis, Registry and Glacier. |
| platform | Includes platform-specific code for Posix and Windows operating systems. (This code is currently unnecessary, but may become necessary if we decide to share bagging and validation code with Windows in the future.) |
| profiles | Contains BagIt profiles supported by Preserv's ingest process. Currently, that's limited to the latest versions of the APTrust and BTR (Beyond the Repository) profiles. The profiles are in DART format, which is richer and more specific than the standard format. DART can convert these back to standard format using its export fuction. This directory also contains the `default.sig` file used by Siegfried, which is the core of the format identification worker. |
| restoration | Contains code used by the Glacier, file, and object restoration workers. |
| scripts | Contains two Ruby scripts. `build.rb` builds the entire suite of Preserv apps, writing the executables into `bin/go-bin`. `test.rb` runs unit, integration, and end-to-end tests, which require spinning up a number of locally running external services. See the [testing documentation](testing.md) for details. |
| testdata | Contains a number of files used in unit/integration/e2e tests. These include files to be bagged, bags to be validated, and JSON files in e2e_results to be matched against the expected outcomes of end-to-end tests. |
| util | Contains utility functions used throughout the codebase. |
| workers | Contains code to harness the contents of the deletion, fixity, ingest, and restoration directories into usable, NSQ-connected workers. These workers are then loaded by the apps in the apps directory. See [Anatomy of a Worker](/workers/anatomy) for details on how these pieces fit together. |
| .env | These are settings files for different environments (dev, test, integration, etc.) See [Settings](settings.md) or the comments in the files themselves for info about which settings are available and what they mean. |
| Dockerfile.build | This file contains instructions for building Docker containers in which to run preservation services. Use `make` to build the containers, as described on the [Docker](docker.md) page. |
| Makefile | Includes commands to build and publish the Docker containers, and to update the CloudFormation template. See [Docker](docker.md) |
| docker-compose.yml | This is an historical artifact used in early, proof-of-concept Docker builds. We may use it as a refence if we move to Kubernetes. |
