+++ 
title = "Advanced_docker_logging"
date = "2020-10-20T12:24:31-07:00"
draft = true
tags = ["docker"]
categories = ["blog"]
authors = ["csgeek"]
+++ 
# Docker Advanced Logging


In the past, I've tried to use GCP logging since it consolidates the logging into a central location.  There are a variety of different logging mechanisms you can use for docker, but for GCP you can read the guide on how to do it on [here](https://cloud.google.com/community/tutorials/docker-gcplogs-driver).  The only downside is that if you do enable it, then you will not be able to see any logs in the console which is big annoyance for me when inspecting things locally.

Many of the alternative drivers that docker can use for logging will disable the console logs.  One exception is the json-file which is what we'll be using. 


In order to avoid this limitation as well as open up the door to a variety of other use cases with logging this tutorial will showcase how to use [Vector](https://vector.dev/docs/) to both manipulate and push logs to various sources. 

:::note
Vector can do a LOT more then just be used to push docker logs but it serves our purposes beautifully.
:::

## File Pattern

I'll go through this pattern first, it's not the recommended pattern IMO but the messages are closer to what you see in the console.

### Requirements:

In this case we need to log the data to a file. You will need to configure the docker application to write to a specific location and then share the file via a docker volume.

For my example use case I was using a telegraf application, so I modified the telegraf app with the following additional setting:

```sh
logfile = "/var/log/telegraf/telegraf.log"
logfile_rotation_interval = "1d"
logfile_rotation_max_size = "20MB"
```

At this point the container will not display anything to STDOUT and instead everything is captured in the log file.  

### Vector Configuration

We need a vector.toml file which drives the behavior of the vector service.

```toml
[sources.telegraf_file]
  # General
  type = "file" # required
  ignore_older = 86400 # optional, no default, seconds
  include = ["/var/log/nginx/*.log"] # required
  start_at_beginning = false # optional, default

  # Priority
  oldest_first = false # optional, default

## Transforms
[transforms.tag_log]
  # General
  type = "add_fields" # required
  inputs = ["telegraf_file"] # required
  overwrite = true # optional, default

  # Fields
  fields.source_id = "22"
  fields.envrionment = "${ENVIRONMENT}"

## Output
[sinks.console]
  type = "console" # required
  inputs = ["tag_log"] # required
  target = "stdout" # optional, default
  encoding.codec = "json" # required
  encoding.timestamp_format = "rfc3339" # optional, default


[sinks.gcp]
  type = "gcp_stackdriver_logs" # required
  inputs = ["tag_log"] # required
  log_id = "telegraf-poller-logs" # required
  credentials_path = "/etc/vector/gcp/staging.json" # optional, no default
  project_id = "esnet-netbeam-develop" # required
  resource.type = "global"
  resource.projectId = "telegraf-poller"
  batch.max_events= 50


```

The vector pipeline will read data from the telegraf file, will apply the transformations defined in `transforms` and finally output the data to the defined sinks.

In this case we're writing out to gcp and console but there are currently 32 output sinks and more are being added every day.  Kafka, PubSub, InfluexDB are just a few of the sinks of note.  

The transform is also tagging the log based on the docker-compose definition.  We're passing certain environment variables, and it's adding the fields.<key> = <value> to the payload.

The value can be anything you'd like but ENVIRONMENT, and Hostname, source_id etc would be useful metadata to tag the log file with if it doesn't already contain that information.


### Docker Compose

```
version: "3.7"
services:
  telegraf:
    image: esnet/telegraf_demo
    depends_on:
      - vector
    volumes:
      - log-demo:/var/log/telegraf
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
  vector:
    image: timberio/vector:0.10.0-alpine
    env_file: .env
    volumes:
      - log-demo:/var/log/telegraf
      - ./logging/:/etc/vector/
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  log-demo:
```

In this case we're logging everything to the volume log-demo and sharing that across the data.  To view the output of the telegraf process.  You'll look at the logs of the vector container rather then telegraf.

Also, keep in mind that the data is persisted, so if you want to start with a clean slate, you will want to remove the old data:

You may look at your current volumes using: `docker volume ls` and remove any of them (after the containers are shutdown) using `docker volume rm <name>` 

Alternatively, `docker system prune` also works.

### ENV file

```sh
NETBEAM_SOURCE_ID=20
ENVIRONMENT=Devel
LOG=trace
```

trace is used mainly for begging, and the rest are just examples.

:::warn
if you use GCP I had to create a directory manually that was missing from the container.


```
FROM timberio/vector:0.10.0-alpine

RUN mkdir /var/lib/vector/ 

ENTRYPOINT ["/usr/local/bin/vector"]
```
:::

## Docker Service Pattern

Rather then relying on every service to create a log file and configuration every application to either redirect stdout, managing volumes and so on.  A much better alternative is to simply leverage the DockerAPI and builtin json-log support.  

As of the writing of this guide, DockerAPI requires either json-file or journald to be set in order to use the DockerAPI.  We'll be doing this using docker-compose which I think is more flexible, but this can be set at the Daemon level if you prefer.  You can read more info on [Vectors Doc](https://vector.dev/docs/reference/sources/docker/).

### Requirements

Unlike the previous attempt, we don't have to modify how the process logs data, but we do have to add a small change to each service to ensure json-file logging is enabled.  For each service the following snippet needs to be added.


```yaml
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### Vector Configuration

vector.toml

```toml
[sources.docker]
  type = "docker" # required
  auto_partial_merge = true # optional, default
  include_containers = ["snmp-telegraf"] # optional, no default

## Transforms
[transforms.tag_log]
  # General
  type = "add_fields" # required
  inputs = ["docker"] # required
  overwrite = true # optional, default

  # Fields
  fields.source_id = "${NETBEAM_SOURCE_ID}"
  fields.envrionment = "${ENVIRONMENT}"

## Output
[sinks.console]
  type = "console" # required
  inputs = ["tag_log"] # required
  target = "stdout" # optional, default
  encoding.codec = "json" # required
  encoding.timestamp_format = "rfc3339" # optional, default


[sinks.gcp]
  type = "gcp_stackdriver_logs" # required
  inputs = ["tag_log"] # required
  log_id = "telegraf-poller-logs" # required
  credentials_path = "/etc/vector/gcp/staging.json" # optional, no default
  project_id = "esnet-netbeam-develop" # required
  resource.type = "global"
  resource.projectId = "telegraf-poller"
  batch.max_events= 50
  ```

  Most of this section is the same, the big difference is the sources is now a docker sink.  You read more about the configuration [here](https://vector.dev/docs/reference/sources/docker/) in my case i ensured to give each container a name, and reference them in the vector.toml.  You may also use labels, image names and a few other patterns that allow you to identify the source of the logs.

If you're using multiple docker services you may look at the [filter](https://vector.dev/docs/reference/transforms/filter/) which allows you to identify the container and drive the logic of further transformations.

### Docker Compose

```yaml
version: "3.7"
services:
  telegraf:
    container_name: snmp-telegraf
    image: esnet/telegraf-snmp:latest
    env_file: .env
    network_mode: "host"
    restart: always
    ports:
      - 9000:9000
    depends_on:
      - vector
    volumes:
      - ./config_generate/mibs:/usr/share/snmp/mibs
      - ./telegraf/teleconf:/etc/telegraf/telegraf.d/
      - ./telegraf/gcpconf:/etc/esnet-snmp/
    # Required for vector/DockerAPI to work correctly
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
  vector:
    image: esnet/vector:0.10.0
    env_file: .env
    volumes:
      - ./logging/:/etc/vector/
      - /var/run/docker.sock:/var/run/docker.sock
      - ./telegraf/gcpconf:/etc/vector/gcp/:ro
```

You no longer need a docker volume.  The format of the logs will look a bit different:

Example:

```json
{
    "container_created_at": "2020-10-16T18:44:28.674057919Z",
    "container_id": "915195afb1339b5abd8a338cbef1a418ebee3d8a1a82f2f7b11005bd7b4e6f49",
    "container_name": "snmp-telegraf",
    "envrionment": "Staging",
    "image": "esnet/telegraf-snmp:latest",
    "label": {
        "com": {
            "docker": {
                "compose": {
                    "config-hash": "0175b7a02f321a0c7394c4702c8faea8294349712bff59d9b06b7f9e3e0d4ea6",
                    "container-number": "1",
                    "oneoff": "False",
                    "project": {
                        "config_files": "../../docker-compose.yml",
                        "working_dir": "/home/samir/telegraf"
                    },
                    "service": "telegraf",
                    "version": "1.26.2"
                }
            }
        }
    },
    "message": "2020-10-16T19:15:00Z W! [inputs.snmp] Collection took longer than expected; not complete after interval of 30s",
    "source_id": "10",
    "source_type": "docker",
    "stream": "stderr",
    "timestamp": "2020-10-16T19:15:00.009568272Z"
}

```

### Env File
The environment section is the same.  There are really no requirements adding any ENV values you're referencing and using LOG=trace if you wish to dig deeper in the logging.

```
NETBEAM_SOURCE_ID=20
ENVIRONMENT=Devel
LOG=trace
```

### Retrieving GCP Logs:

For the examples in this tutorial, all logs can be retrieved via this query:

```
resource.type="global"  resource.labels.project_id ="esnet-netbeam-develop"
```

you can further dig deeper into the logs by using GCP query language

```
resource.type="global"  resource.labels.project_id ="esnet-netbeam-develop" jsonPayload.envrionment="Staging" 
```

Among other more [Advanced](https://cloud.google.com/logging/docs/view/advanced-queries) Querying

### Closing Notes

For all of these examples I'm mounting a logging folder which contains the vector.toml file. You can directly mount the file but that was giving me some issues.

If you are pushing to GCP you naturally also need to expose the service json key and paths to allow it to authenticate.

You may also use ENV variables in vector.toml.  Every value may be a ENV config.  So a more flexible pattern would be to use something like this:


```toml
[sinks.gcp]
  type = "gcp_stackdriver_logs" # required
  inputs = ["tag_log"] # required
  log_id = "${LOG_NAME}" # required
  credentials_path = "/etc/vector/gcp/${GCP_KEY}" # optional, no default
  project_id = "${GCP_PROJECT_ID}" # required
  resource.type = "global"
  resource.projectId = "telegraf-poller"
  batch.max_events= 5
```
