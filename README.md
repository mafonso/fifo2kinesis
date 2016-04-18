# FIFO to Kinesis Pipeline

[![Build Status](https://travis-ci.com/acquia/fifo2kinesis.svg?token=PH71WkhMufTnsVvCU5rV&branch=master)](https://travis-ci.com/acquia/fifo2kinesis)

This app continuously reads data from a named pipe (FIFO) and publishes it
to a Kinesis stream.

## Installation

This project uses the [GB build tool](https://getgb.io/). Assuming that GB
is installed, run the following command in the project's root directory to
build the `fifo2kinesis` binary:

```shell
gb build
```

## Usage

Create a named pipe:

```shell
mkfifo ./kinesis.pipe
```

Run the app:

```shell
./bin/fifo2kinesis --fifo-name=$(pwd)/kinesis.pipe --stream-name=mystream
```

Write to the FIFO:

```shell
echo "Streamed at $(date)" > kinesis.pipe
```

The line will be published to the `mystream` Kinesis stream.

### Configuration

Configuration is read from command line options and environment variables
in that order of precedence. The following options and env variables are
available:

* `--fifo-name`, `FIFO2KINESIS_FIFO_NAME`: The absolute path of the named pipe.
* `--stream-name`, `FIFO2KINESIS_STREAM_NAME`: The name of the Kinesis stream.
* `--partition-key`, `FIFO2KINESIS_PARTITION_KEY`: The partition key, a random string if omitted.
* `--buffer-queue-limit`, `FIFO2KINESIS_BUFFER_QUEUE_LIMIT`: The number of items that trigger a buffer flush.
* `--flush-interval`, `FIFO2KINESIS_FLUSH_INTERVAL`: The number of seconds before the buffer is flushed.
* `--flush-handler`, `FIFO2KINESIS_FLUSH_HANDLER`: Defaults to "kinesis", use "logger" for debugging.
* `--debug`, `FIFO2KINESIS_DEBUG`: Show debug level log messages.

The application also requires credentials to publish to the specified
Kinesis stream. It uses the same [configuration mechanism](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#config-settings-and-precedence)
as the AWS CLI tool, minus the command line options.

### Running With Upstart

The application exits immediately with a non-zero exit code on all AWS
errors. Therefore it is useful to run the app with an event system such as
Upstart in order to respawn the service when an error occurs.

```
description "FIFO to Kinesis Pipeline"
start on runlevel [2345]

respawn
respawn limit 3 30
post-stop exec sleep 5

exec /path/to/fifo2kinesis --fifo-name=/path/to/named.pipe --stack-name=mystack
```

### Publishing Logs From Syslog NG

**Disclaimer**: [fluentd](http://www.fluentd.org/) is probably a better
option for sending logs to Kinesis.

Syslog NG provides the capability to use a named pipe as a destination. Use
this app to read log messages from the FIFO and publish them to a Kenisis
stream.

On Ubuntu 14.04, create a file named `/etc/syslog-ng/conf.d/01-kinesis.conf`
with the following configration:

```
destination d_pipe { pipe("/var/syslog.pipe"); };
log { source(s_src); destination(d_pipe); };
```

Make a FIFO:

```
mkfifo /var/syslog.pipe
```

Start the app:

```
./bin/fifo2kinesis --fifo-name=/var/syslog.pipe --stream-name=mystream
```

Restart syslog-ng:

```
service syslog-ng restart
```

All log messages will be published to Kinesis.
