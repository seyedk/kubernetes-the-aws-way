# Prerequisites

## Amazon AWS

This tutorial leverages the [Amazon aWS](https://aws.amazon.com/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://aws.amazon.com/free/) for $100 in free credits.

[Estimated cost](https://calculator.s3.amazonaws.com/index.html) to run this tutorial: $2.22 per hour ($15.39 per day).

> The compute resources required for this tutorial exceed the AWS  free tier.

## AWS CLI

### Install the AWS CLI 

Follow the AWS [documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) to install and configure the `aws` command line utility.

Verify the Google Cloud SDK version is `aws-cli/1.15.66 Python/3.7.0 Darwin/18.2.0 botocore/1.10.65` or higher:

```
aws --version
```

### Set a Default Compute Region and Zone

This tutorial assumes a default compute region and zone have been configured.

If you are using the `aws` command-line tool for the first time `configure` is the easiest way to do this:

```
aws configure
```

O

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with `synchronize-panes` enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable `synchronize-panes`: `ctrl+b` then `shift :`. Then type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

> To Split the window to multiple (most cases 3 ) panes: `ctrl+b` then press double quote mark (`"`). Then to adjust the layout of the panges: `ctrl+b` then `spacebar`

Next: [Installing the Client Tools](02-client-tools.md)
