# Telegraf - A native agent for InfluxDB [![Circle CI](https://circleci.com/gh/influxdb/telegraf.svg?style=svg)](https://circleci.com/gh/influxdb/telegraf)

Telegraf is an agent written in Go for collecting metrics from the system it's
running on or from other services and writing them into InfluxDB.

Design goals are to have a minimal memory footprint with a plugin system so
that developers in the community can easily add support for collecting metrics
from well known services (like Hadoop, or Postgres, or Redis) and third party
APIs (like Mailchimp, AWS CloudWatch, or Google Analytics).

We'll eagerly accept pull requests for new plugins and will manage the set of
plugins that Telegraf supports. See the bottom of this doc for instructions on
writing new plugins.

## Quickstart

* Build from source or download telegraf:

### Linux packages for Debian/Ubuntu and RHEL/CentOS:

NOTE: version 0.1.4+ has introduced some breaking changes! A 0.1.4+ telegraf
agent is NOT backwards-compatible with a config file from 0.1.3 and below.
That being said, the difference is not huge, see below for an example on
how to setup the new config file.

As well, due to a breaking change to the InfluxDB integer line-protocol, there
are some InfluxDB compatibility requirements:

* InfluxDB 0.9.3+ (including nightly builds) requires Telegraf 0.1.5+
* InfluxDB 0.9.2 and prior requires Telegraf 0.1.4

Latest:
* http://get.influxdb.org/telegraf/telegraf_0.1.8_amd64.deb
* http://get.influxdb.org/telegraf/telegraf-0.1.8-1.x86_64.rpm

Binaries:
* http://get.influxdb.org/telegraf/telegraf_linux_amd64_0.1.8.tar.gz
* http://get.influxdb.org/telegraf/telegraf_linux_386_0.1.8.tar.gz
* http://get.influxdb.org/telegraf/telegraf_linux_arm_0.1.8.tar.gz

0.1.4:
* http://get.influxdb.org/telegraf/telegraf_0.1.4_amd64.deb
* http://get.influxdb.org/telegraf/telegraf-0.1.4-1.x86_64.rpm

### OSX via Homebrew:

```
brew update
brew install telegraf
```

### From Source:

Telegraf manages dependencies via `godep`, which gets installed via the Makefile
if you don't have it already. You also must build with golang version 1.4+

1. [Install Go](https://golang.org/doc/install)
2. [Setup your GOPATH](https://golang.org/doc/code.html#GOPATH)
3. run `go get github.com/influxdb/telegraf`
4. `cd $GOPATH/src/github.com/influxdb/telegraf`
5. run `make`

### How to use it:

* Run `telegraf -sample-config > telegraf.toml` to create an initial configuration
* Edit the configuration to match your needs
* Run `telegraf -config telegraf.toml -test` to output one full measurement sample to STDOUT
* Run `telegraf -config telegraf.toml` to gather and send metrics to configured outputs.
* Run `telegraf -config telegraf.toml -filter system:swap`
to enable only the system & swap plugins defined in the config.

## Telegraf Options

Telegraf has a few options you can configure under the `agent` section of the
config. If you don't see an `agent` section run
`telegraf -sample-config > telegraf.toml` to create a valid initial
configuration:

* **hostname**: The hostname is passed as a tag. By default this will be
the value retured by `hostname` on the machine running Telegraf.
You can override that value here.
* **interval**: How ofter to gather metrics. Uses a simple number +
unit parser, ie "10s" for 10 seconds or "5m" for 5 minutes.
* **debug**: Set to true to gather and send metrics to STDOUT as well as
InfluxDB.

## Plugin Options

There are 5 configuration options that are configurable per plugin:

* **pass**: An array of strings that is used to filter metrics generated by the
current plugin. Each string in the array is tested as a prefix against metric names
and if it matches, the metric is emitted.
* **drop**: The inverse of pass, if a metric name matches, it is not emitted.
* **tagpass**: (added in 0.1.5) tag names and arrays of strings that are used to filter metrics by
the current plugin. Each string in the array is tested as an exact match against
the tag name, and if it matches the metric is emitted.
* **tagdrop**: (added in 0.1.5) The inverse of tagpass. If a tag matches, the metric is not emitted.
This is tested on metrics that have passed the tagpass test.
* **interval**: How often to gather this metric. Normal plugins use a single
global interval, but if one particular plugin should be run less or more often,
you can configure that here.

### Plugin Configuration Examples

This is a full working config that will output CPU data to an InfluxDB instance
at 192.168.59.103:8086, tagging measurements with dc="denver-1". It will output
measurements at a 10s interval and will collect totalcpu & percpu data.

```
[tags]
    dc = "denver-1"

[agent]
    interval = "10s"

# OUTPUTS
[outputs]
[outputs.influxdb]
    url = "http://192.168.59.103:8086" # required.
    database = "telegraf" # required.

# PLUGINS
[cpu]
    percpu = true
    totalcpu = true
```

Below is how to configure `tagpass` and `tagdrop` parameters (added in 0.1.5)

```
# Don't collect CPU data for cpu6 & cpu7
[cpu.tagdrop]
    cpu = [ "cpu6", "cpu7" ]

[disk]
[disk.tagpass]
    # tagpass conditions are OR, not AND.
    # If the (filesystem is ext4 or xfs) OR (the path is /opt or /home)
    # then the metric passes
    fstype = [ "ext4", "xfs" ]
    path = [ "/opt", "/home" ]
```

## Supported Plugins

**You can view usage instructions for each plugin by running**
`telegraf -usage <pluginname>`

Telegraf currently has support for collecting metrics from

* disque
* elasticsearch
* exec (generic JSON-emitting executable plugin)
* haproxy
* httpjson (generic JSON-emitting http service plugin)
* kafka_consumer
* leofs
* lustre2
* memcached
* mongodb
* mysql
* nginx
* postgresql
* prometheus
* rabbitmq
* redis
* rethinkdb
* system (mem, CPU, load, etc.)

We'll be adding support for many more over the coming months. Read on if you
want to add support for another service or third-party API.

## Output options

Telegraf also supports specifying multiple output sinks to send data to,
configuring each output sink is different, but examples can be
found by running `telegraf -sample-config`

## Supported Outputs

* influxdb
* kafka
* datadog

## Contributing

Please see the
[contributing guide](https://github.com/influxdb/telegraf/blob/master/CONTRIBUTING.md)
for details on contributing a plugin or output to Telegraf

## Testing

### Execute short tests

execute `make test-short`

### Execute long tests

As Telegraf collects metrics from several third-party services it becomes a
difficult task to mock each service as some of them have complicated protocols
which would take some time to replicate.

To overcome this situation we've decided to use docker containers to provide a
fast and reproducible environment to test those services which require it.
For other situations
(i.e: https://github.com/influxdb/telegraf/blob/master/plugins/redis/redis_test.go )
a simple mock will suffice.

To execute Telegraf tests follow these simple steps:

- Install docker compose following [these](https://docs.docker.com/compose/install/)
instructions
    - mac users should be able to simply do `brew install boot2docker`
      and `brew install docker-compose`
- execute `make test`

### Unit test troubleshooting

Try cleaning up your test environment by executing `make test-cleanup` and
re-running
