# FIFO to Kinesis Pipeline

[![Build Status](https://travis-ci.com/acquia/fifo2kinesis.svg?token=PH71WkhMufTnsVvCU5rV&branch=master)](https://travis-ci.com/acquia/fifo2kinesis)

This app continuously reads data from a [named pipe (FIFO)](https://en.wikipedia.org/wiki/Named_pipe)
and publishes it to a [Kinesis](https://aws.amazon.com/kinesis/) stream.

## Why?

FIFOs are a great way to send data from one application to another. Having
an open pipe that ships data to Kinesis facilitates a lot of interesting use
cases. One such example is using the named pipe support in
[rsyslog](http://www.rsyslog.com/doc/v8-stable/configuration/modules/ompipe.html)
and [syslog-ng](https://www.balabit.com/sites/default/files/documents/syslog-ng-ose-latest-guides/en/syslog-ng-ose-guide-admin/html/configuring-destinations-pipe.html)
to send log streams to Kinesis.

Admittedly, it would be really easy to write a handful of lines of code in
a bash script using the AWS CLI to achieve the same result, however the
fifo2kinesis app buffers data read from the FIFO, batch-publishes it to
Kinesis, and leverages a persistent failure handling mechanism in order to
efficiently and reliably process large data streams.

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

The line will be published to the `mystream` Kinesis stream within the
default flush interval of 5 seconds.

#### Quick start for the impatient among us

If you are impatient like me and want your oompa loompa now, modify the
`--buffer-queue-limit`, `--flush-interval`, and `--flush-handler` options so
that what you send to the FIFO is written to STDOUT immediately instead of a
buffered write to Kinesis. This doesn't do much, but it provides immediate
gratification and shows how the app works when you play with the options.

```shell
./bin/fifo2kinesis --fifo-name=$(pwd)/kinesis.pipe --buffer-queue-limit=1 --flush-interval=0 --flush-handler=logger
```

### Configuration

Configuration is read from command line options and environment variables
in that order of precedence. The following options and env variables are
available:

* `--fifo-name`, `FIFO2KINESIS_FIFO_NAME`: The absolute path of the named pipe.
* `--stream-name`, `FIFO2KINESIS_STREAM_NAME`: The name of the Kinesis stream.
* `--partition-key`, `FIFO2KINESIS_PARTITION_KEY`: The partition key, a random string if omitted.
* `--buffer-queue-limit`, `FIFO2KINESIS_BUFFER_QUEUE_LIMIT`: The number of items that trigger a buffer flush.
* `--failed-attempts-dir`, `FIFO2KINESIS_FAILED_ATTEMPTS_DIR`: The directory that logs failed attempts for retry.
* `--flush-interval`, `FIFO2KINESIS_FLUSH_INTERVAL`: The number of seconds before the buffer is flushed.
* `--flush-handler`, `FIFO2KINESIS_FLUSH_HANDLER`: Defaults to "kinesis", use "logger" for debugging.
* `--debug`, `FIFO2KINESIS_DEBUG`: Show debug level log messages.

The application also requires credentials to publish to the specified
Kinesis stream. It uses the same [configuration mechanism](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#config-settings-and-precedence)
as the AWS CLI tool, minus the command line options.

### Running With Upstart

```
description "FIFO to Kinesis Pipeline"
start on runlevel [2345]

respawn
respawn limit 3 30
post-stop exec sleep 5

exec /path/to/fifo2kinesis --fifo-name=/path/to/named.pipe --stack-name=mystack
```

### Publishing Logs From Syslog NG

**Disclaimer**: You should take a look at [fluentd](http://www.fluentd.org/).
You won't find an argument in this README as to why you should choose one
over the other. I want to make sure you have all the options in front of you
so that you can make the best decision for your specific use case.

Syslog NG provides the capability to use a named pipe as a destination. Use
fifo2kinesis to read log messages from the FIFO and publish them Kenisis.

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

The log stream will now be published to Kinesis.
