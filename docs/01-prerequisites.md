# Prerequisites

## Amazon Web Services Platform

> The compute resources required for this tutorial exceed the Amazon Web Services free tier.

## Amazon Web Services CLI

### Install the Amazon Web Services CLI

Follow the Amazon Web Services CLI [documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) to install and configure the `aws` command line utility.


### Set a Default Compute Region and Zone

This tutorial assumes a default compute region and zone have been configured.

If you are using the `aws` command-line tool for the first time `configure` is the easiest way to do this:

```
aws configure
```

Otherwise set a default compute region:

```
gcloud config set compute/region us-west1
```

Set a default compute zone:

```
gcloud config set compute/zone us-west1-c
```

> Use the `gcloud compute zones list` command to view additional regions and zones.

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with `synchronize-panes` enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable `synchronize-panes`: `ctrl+b` then `shift :`. Then type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
