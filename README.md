# gha-runner
A simple GitHub Action for creating self-hosted runners. Currently, this only supports AWS and x86_64 Linux AMIs. This runner is heavily inspired by [ec2-github-runner](https://github.com/machulav/ec2-github-runner) but rewritten to support additional cloud providers and to better meet some needs of the OMSF community.

## A note on security before using this action
The following is a note from GitHub on using self-hosted runners on public repos.

> We recommend that you do not use self-hosted runners with public repositories.
>
> Forks of your public repository can potentially run dangerous code on your self-hosted runner machine by creating a pull request that executes the code in a workflow.

For more information see the [self-hosted runner security docs](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners#self-hosted-runner-security-with-public-repositories). As well as some good [community recommendations and pitfalls](https://github.com/orgs/community/discussions/26722). We _do not_ recommend running these pipelines on a PR.

## Cloud Setup Docs
- [AWS](docs/aws.md)

## Example Setups
- [openmm-gpu-test](https://github.com/omsf-eco-infra/openmm-gpu-test) - a simple GPU for validating OpenMM on AWS

## Inputs

### Shared Inputs
| Input    | Description                                                                | Required | Default |
|----------|----------------------------------------------------------------------------|----------|---------|
| action   | Whether to start or stop. Options: "start", "stop"                         | true     |         |
| provider | The cloud provider to use to provision a runner. Will not start if not set.| true     |         |
| repo     | The repo to run against. Will use the current repo if not specified.       | false    | The repo the runner is running in |

### AWS `start` Inputs
| Input                 | Description                                                                                                        | Required for start | Default |
|-----------------------|--------------------------------------------------------------------------------------------------------------------|------------------- |---------|
| aws_home_dir          | The AWS AMI home directory to use for your runner. Will not start if not specified.                                | true               |         |
| aws_iam_role          | The optional AWS IAM role to assume for provisioning your runner.                                                  | false              |         |
| aws_image_id          | The machine AMI to use for your runner. This AMI can be a default but should have docker installed in the AMI.     | true               |         |
| aws_instance_type     | The type of instance to use for your runner. For example: t2.micro, t4g.nano, etc. Will not start if not specified.| true               |         |
| aws_region_name       | The AWS region name to use for your runner. Will not start if not specified.                                       | true               |         |
| aws_root_device_size  | The root device size in GB to use for your runner.                                                                 | false              | The default AMI root device size |
| aws_security_group_id | The AWS security group ID to use for your runner. Will use the account default security group if not specified.    | false              | The default AWS security group |
| aws_subnet_id         | The AWS subnet ID to use for your runner. Will use the account default subnet if not specified.                    | false              | The default AWS subnet ID |
| aws_tags              | The AWS tags to use for your runner, formatted as a JSON list. See `README` for more details.                      | false              |         |
| extra_gh_labels       | Any extra GitHub labels to tag your runners with. Passed as a comma-separated list with no spaces.                 | false              |         |
| instance_count        | The number of instances to create, defaults to 1                                                                   | false              | 1       |
| gh_timeout            | The timeout in seconds to wait for the runner to come online as seen by the GitHub API. Defaults to 1200 seconds.  | false              | 1200    |

### AWS `stop` Inputs
| Input             | Description                                                                                                                                                         | Required for stop| Default | Note |
|-------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------- |----------------- |---------| ---- |
| instance_mapping  | A JSON object mapping instance ids to unique GitHub runner labels This should be the same as the AWS `start` output, `mapping`. Required to stop created instances. | true             |         | This should be the mapping output as generated by the `start` run. |

## Outputs
| Name | Description |
| ---- | ----------- |
| mapping | A JSON object mapping instance IDs to unique GitHub runner labels. This is used in conjunction with the `instance_mapping` input when stopping. |
| instances | A JSON list of the GitHub runner labels to be used in the 'runs-on' field |


## Testing the action in development
Testing the action was primarily done using [nektos/act]("https://github.com/nektos/act") to test locally. To do a test run with some basic defaults:
```yaml
name: Testing
on: [push]

jobs:
  hello_world_job:
    runs-on: ubuntu-latest
    name: Test the gha_runner
    steps:
      # To use this repository's private action,
      # you must check out the repository
      - name: Checkout
        uses: actions/checkout@v4
      - name: Local runner run
        id: aws_start
        uses: ./ # Uses an action in the root directory
        with:
          repo: "testing/testing"
          provider: "aws"
          action: "start"
          aws_image_id: <use an image here that has docker installed>
          aws_instance_type: t2.micro
          instance_count: 2
          aws_region_name: us-east-1
          aws_home_dir: /home/ec2-user
          aws_labels: testing
        env:
          GH_PAT: ${{ secrets.GH_PAT }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      - name: Stop instances
        uses: ./
        with:
          repo: "testing/testing"
          provider: "aws"
          action: "stop"
          instance_mapping: ${{ steps.aws_start.outputs.mapping }}
          aws_region_name: us-east-1
        env:
          GH_PAT: ${{ secrets.GH_PAT }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}


```
_NOTE: For guidelines on setting secrets view the act docs found [here](https://nektosact.com/usage/index.html?highlight=Secrets#secrets)_
This file was placed in a `test-worklows` directory.

Then it was run using the following:
```sh
act -W test-workflows/ --verbose
```

For testing on M-series Macs, use the following command instead:
```sh
act --container-architecture linux/arm64 -W test-workflows/ --verbose
```

## Acknowledgements
This action was heavily inspired by the [ec2-github-runner](https://github.com/machulav/ec2-github-runner). This action takes much of its structure and inspiration around architecture from the `ec2-github-runner` itself. Thank you to the authors of that action for providing a solid foundation to build upon.
