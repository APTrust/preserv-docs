# Docker

All Preserv components run in Docker containers in Amazon's Fargate service. All except the two cron jobs, `apt_queue` and `apt_queue_fixity`, scale automatically based on ECS triggers.

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
