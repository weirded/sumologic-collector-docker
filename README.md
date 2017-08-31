# Sumo Logic Collector for Docker

This repository offers several variants of Docker images to run the Sumo Logic Collector. When images are run, the Collector automatically registers with the Sumo Logic service and create sources based on a `sumo-sources.json` file. The Collector is configured [ephemeral](https://help.sumologic.com/Send_Data/Installed_Collectors/sumo.conf).

### Configuration

Log into Sumo Logic to create an Access ID and an Access Key to register the Sumo Logic Collector. See the [online help](https://help.sumologic.com/Manage/Security/Access-Keys) for instructions.

The Sumo Logic Collector can be configured either with environment variables, or a volume mounted `user.properties` file.

#### Approach 1: Environment variables

The following environment variables are supported. These can be passed to the `docker run` command with the `-e` flag.

* `SUMO_ACCESS_ID` - Can be used to pass the access ID instead of passing it in as a commandline argument.
* `SUMO_ACCESS_KEY` - Can be used to pass the access key instead of passing it in as a commandline argument.
* `SUMO_COLLECTOR_NAME` - Allows configuring the name of the Collector. The default is set dynamically to the value within `/etc/hostname`.
* `SUMO_COLLECTOR_NAME_PREFIX` - Allows configuring a prefix to the collector name. Useful when overriding `SUMO_COLLECTOR_NAME` with the docker hostname
* `SUMO_SOURCES_JSON` - Allows specifying the path of the `sumo-sources.json` file. The default is `/etc/sumo-sources.json`.
* `SUMO_SYNC_SOURCES` - If `true` SUMO_SOURCES_JSON file(s) will be continuously monitored and synchronized with the Collector's configuration. This will also disable editing of the collector in the Sumo UI. default `false`.
* `SUMO_PROXY_HOST` - Sets proxy host when a proxy server is used.
* `SUMO_PROXY_PORT` - Sets proxy port when a proxy server is used.
* `SUMO_PROXY_USER` - Sets proxy user when a proxy server is used with authentication.
* `SUMO_PROXY_PASSWORD` - Sets proxy password when a proxy server is used with authentication.
* `SUMO_PROXY_NTLM_DOMAIN` - Sets proxy NTLM domain when a proxy server is used with NTLM authentication.
* `SUMO_CLOBBER` - When true, if there is any existing Collector with the same name, that Collector will be deleted. default is `false`
* `SUMO_DISABLE_SCRIPTS` - If your organization's internal policies restrict the use of scripts, you can disable the creation of script-based Script Sources. When this parameter is passed, this option is removed from the Sumo Logic Web Application, and Script Source cannot be configured. default is `false`
* `SUMO_JAVA_MEMORY_INIT` - Sets the initial java heap size (in MB). Default: `64`.
* `SUMO_JAVA_MEMORY_MAX` - Sets the maximum java heap size (in MB). Default: `128`.

#### Approach 2: User.properties File
Alternatively, you can provide a `user.properties` file via a Docker volume mount.  See the [online help](http://help.sumologic.com/Send_Data/Installed_Collectors/05Reference_Information_for_Collector_Installation/06user.properties) for a list of possible parameters.

To use a custom `user.properties` file, you must pass the environment variable `SUMO_GENERATE_USER_PROPERTIES=false`, as well as providing the Docker volume mount to replace the file located at `/opt/SumoCollector/config/user.properties`.

Example:

```bash
docker run <other options> -e SUMO_GENERATE_USER_PROPERTIES=false -v $some_path/user.properties:/opt/SumoCollector/config/user.properties collector:$tag
```

### Variants

#### Docker Collection

Images tagged with `latest` or `latest-docker-sources` are available for Docker collection. When run, the Collector listens on the docker unix socket for container logs, events and stats. Plug your access ID and an access key into the commandline below:

```bash
docker run -d -v /var/run/docker.sock:/var/run/docker.sock --name="sumo-logic-collector"  sumologic/collector:latest <Access ID> <Access key>
```

#### Syslog Collection

A simple "batteries included" syslog image is available and tagged `latest-syslog`. When run, the Collector listens on port 514 TCP and UDP for syslog traffic. Simply plug your access ID and an access key into the commandline below:


```bash
docker run -d -p 514:514 -p 514:514/udp --name="sumo-logic-collector" sumologic/collector:latest-syslog [Access ID] [Access key]
```

#### File Collection

Another "batteries included" image is available and tagged `latest-file`. When run, the Collector collects all files from `/tmp/clogs/`. Docker volumes need to be used to make logs available in this directory. Plug your credentials into the commandline below and adjust the volume options as needed:

```bash
docker run -v /tmp/clogs:/tmp/clogs -d --name="sumo-logic-collector" sumologic/collector:latest-file [Access ID] [Access key]
```

Using the `/etc/sumo-containers.json` source file you can collect logs from all containers.

```bash
docker run -v /var/lib/docker/containers:/var/lib/docker/containers:ro -d --name="sumo-logic-collector" -e SUMO_SOURCES_JSON=/etc/sumo-containers.json sumologic/collector:latest-file [Access ID] [Access key]
```


#### Custom Configuration

A base image to build your own image with a custom configuration is tagged `latest-no-source`. You need to add  `/etc/sumo-sources.json` to run it.
Examples are available in `example` [in GitHub](https://github.com/SumoLogic/sumologic-collector-docker/tree/master/example), along with some example configuration files. Pick one of the examples and rename to `sumo-sources.json` or create one from scratch. See  our [online help](https://help.sumologic.com/Send_Data/Sources/Use_JSON_to_Configure_Sources) for more details.

After configuring a `sumo-sources.json` file, create a `Dockerfile` similar to the one below:

```
FROM sumologic/collector:latest-no-source
MAINTAINER Happy Sumo Customer
ADD sumo-sources.json /etc/sumo-sources.json
```

Build an image with your configuration:

```bash
docker build --tag="yourname/sumocollector" .
```

To run your image, plug your access ID and an access key into the commandline below to run the container:

```bash
docker run -d --name="sumo-logic-collector" yourname/sumocollector [Access ID] [Access key]
```

Depending on the source setup, additional commandline parameters will be needed to create container.


#### Source Templates

This container supports source json configuration templates allowing for string substitution using environment variables. This works by finding all files with a .json.tmpl extentions, looping through all environment variables and replacing the values. Finally the file is renamed to .json.

NOTE: You can also create your own docker image with the tmpl files embedded rather than a volume mount.

For example, if the container was started with the following environment variables and file /etc/sumo-containers.json.tmpl:

```
docker run -v /var/lib/docker/containers:/var/lib/docker/containers:ro -v /path/to/sources:/sumo  -d --name="sumo-logic-collector" -e SUMO_SOURCES_JSON=/sumo/sources.json -e ENVIRONMENT=prod sumologic/collector:latest-file [Access ID] [Access key]
```

File /path/to/sources/sources.json.tmpl

```
{
  "api.version": "v1",
  "sources": [
    {
      "sourceType" : "LocalFile",
      "name": "localfile-collector-container",
      "pathExpression": "/var/lib/docker/containers/**/*.log",
      "category": "${ENVIRONMENT}/containers"
    }
  ]
}
```

The resulting output of /sumo/sources.json will be
```
{
  "api.version": "v1",
  "sources": [
    {
      "sourceType" : "LocalFile",
      "name": "localfile-collector-container",
      "pathExpression": "/var/lib/docker/containers/**/*.log",
      "category": "prod/containers"
    }
  ]
}
```
