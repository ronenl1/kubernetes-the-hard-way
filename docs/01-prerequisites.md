# Prerequisites

## Amazon Web Services Platform

> The compute resources required for this tutorial can be used by Amazon Web Services free tier account.

## Amazon Web Services CLI

### Install the Amazon Web Services CLI

Follow the Amazon Web Services CLI [documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) to install and configure the `aws` command line utility.


### Set a Default Compute Region and Zone

This tutorial assumes a default compute region and zone have been configured.

If you are using the `aws` command-line tool for the first time `configure` is the easiest way to do this:

```
aws configure
```

Then fill the Access Key ID and Secret Access Key of your account:

```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

Set the default compute zone and default output format:

```
Default region name [None]: eu-west-1
Default output format [None]: json
```

Next: [Installing the Client Tools](02-client-tools.md)
