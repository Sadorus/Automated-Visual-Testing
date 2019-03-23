# AET Docker support
<p align="center">
  <img src="https://raw.githubusercontent.com/Skejven/aet-docker/master/misc/aet-docker.png" alt="AET Docker Logo"/>
</p>

Contains Dockerfiles and example docker swarm configuration to setup AET instance.

## Docker Images

### AET ActiveMq
Hosts [Apache ActiveMQ](http://activemq.apache.org/) that is used as the communication bus by the AET components.
### AET Browsermob
Hosts BrowserMob proxy that is used by AET to collect status codes and inject headers to requests.
### AET Apache Karaf 
Hosts [Apache Karaf](https://karaf.apache.org/) OSGi applications container.
It contains all AET modules (bundles): Runner, Workers, Web-API, Datastorage, Executor, Cleaner and runs them within OSGi context.
### AET Apache Server 
Runs [Apache Server](https://httpd.apache.org/) that hosts [AET Report](https://github.com/Cognifide/aet/wiki/SuiteReport)
and [AET suite generator](https://github.com/m-suchorski/suite-generator/tree/feature/suite)

## Example AET instance with docker swarm
Example Docker Swarm cluster that consists of only 1 node/manager.
It enables to run fully functional AET instance that consists of:
- MongoDB container with mounted volume (for persistency)
- Selenium Grid with Hub and 3 Nodes (2 Chrome instances each, totally 6 browsers)
- AET ActiveMq container
- AET Browsermob container
- AET Apache Karaf container with AET core installed (Runner, Workers, Web-API, Datastorage, Executor)
- AET Apache Server container with AET Report and [AET suite generator](https://github.com/m-suchorski/suite-generator/tree/feature/suite)

Read more about that example setup in the [Example Swarm Readme](https://github.com/Skejven/aet-docker/tree/master/example-aet-swarm).

### Prerequisites
- Docker installed on your host (either ["Docker"](https://docs.docker.com/install/) (e.g. Docker for Windows) 
or ["Docker Tools"](https://docs.docker.com/toolbox/overview/)).
- Docker swarm initialized. 
See this [swarm-tutorial: create swarm](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/) for detailed instructions.
  - **TLDR;** If you are using:
    - *Docker*: Run command: `docker swarm init`.
    - *Docker Tools*: Run `docker swarm init --advertise-addr <manager-ip>` where `<manager-ip>` 
is the IP of your docker-machine (usually `192.168.99.100`).

***
If you are using ["Docker Tools"](https://docs.docker.com/toolbox/overview/) and docker-machine please
create and mount your `AET_ROOT` directory to the Virtual Machine first. One of the ways to do this using VM GUI:
  1. Start "Oracle VM VirtualBox Manager"
  2. Right-Click `<machine name>` (*default*)
  3. Settings...
  4. Shared Folders
  5. The Folder+ Icon on the Right (Add Share)
  6. Folder Path: `<host dir>` (path to `AET_ROOT` on your host, e.g. `c:/Workspace/example-aet-swarm`)
  7. Folder Name: `<mount name>` (e.g. `osgi-configs`)
  8. Check on "Auto-mount" and "Make Permanent"
  9. Restart `<machine name>` (*default*) to apply changes.
Now you should see `osgi-configs` on your docker-machine VM.
You may check it by invoking:
`docker-machine ssh default "ls /osgi-configs"` it should list:
```
aet-swarm.yml
configs
```
***

### Running example instance
> Notice - this instruction guides you how to setup AET instance using single-node swarm cluster. 
> *This is not production recommended setup!*
1. Download `example-aet-swarm.zip` from the [release](https://github.com/Skejven/aet-docker/releases).
2. Unzip the files to the folder from where docker stack will be deployed (from now on we will call it `AET_ROOT`).
Contents of the `AET_ROOT` directory should look like:
```
├── aet-swarm.yml
├── configs
│   ├── com.cognifide.aet.cleaner.CleanerScheduler-main.cfg
│   ├── com.cognifide.aet.proxy.RestProxyManager.cfg
│   ├── com.cognifide.aet.queues.DefaultJmsConnection.cfg
│   ├── com.cognifide.aet.rest.helpers.ReportConfigurationManager.cfg
│   ├── com.cognifide.aet.runner.MessagesManager.cfg
│   ├── com.cognifide.aet.runner.RunnerConfiguration.cfg
│   ├── com.cognifide.aet.vs.mongodb.MongoDBClient.cfg
│   ├── com.cognifide.aet.worker.drivers.chrome.ChromeWebDriverFactory.cfg
│   └── com.cognifide.aet.worker.listeners.WorkersListenersService.cfg
```
  - If you are using docker-machine (otherwise ignore this point)
   you should change `aet-swarm.yml` `volumes` section for the `karaf` service to:
    ```yaml
        volumes:
          - /osgi-configs/configs:/configs # when using docker-machine, use mounted folder
    ```
3. From the `AET_ROOT` run `docker stack deploy -c aet-swarm.yml aet`.
4. Wait about 1-2 minutes until Karaf start finishes.

When it is ready, you should see the information in the [Karaf console](http://localhost:8181/system/console/bundles) (credentials: `karaf/karaf`):

  > Bundle information: 204 bundles in total - all 204 bundles active

You may also check the status of Karaf by executing

```bash
docker ps --format "table {{.Image}}\t{{.Status}}" --filter expose=8181/tcp
```

When you see status `healthy` it means Karaf is running correctly

> IMAGE                     STATUS
> skejven/aet_karaf:0.4.0   Up 20 minutes (healthy)

### Running AET Suite
To run AET Suite simply define `endpointDomain` to AET Karaf IP with `8181` port, e.g.:
> `./aet.sh http://localhost:8181`
or
> ` mvn aet:run -DendpointDomain=http://localhost:8181`

Read more about running AET suite [here](https://github.com/Cognifide/aet/wiki/RunningSuite).

## Best practice when setting up AET instance
1. Control changes in `aet-swarm.yml` and config files over time! Use version control system (e.g. [GIT](https://git-scm.com/)) to keep tracking changes of `AET_ROOT` contents.
2. If you value your data - reports results and history of running suites, remember about **backing up MongoDB volume**. If you use external MongoDB, also back up its `/data/db` regularly!
3. Provide at least [minimum requirements](#minimum-requirements) machine for your docker cluster.

## Instance configuration
Thanks to the mounted OSGi configs you may now configure
instance via `AET_ROOT/configs` configuration files. Read more about possible configurations
in the [example swarm config section](https://github.com/Skejven/aet-docker/tree/master/example-aet-swarm#configs).

## Building
### Prerequisites
- Docker installed on your host.

1. Clone this repository
2. Build all images using `build.sh {tag}`.
You should see following images:
```
    skejven/aet_report:{tag}
    skejven/aet_karaf:{tag}
    skejven/aet_browsermob:{tag}
    skejven/aet_activemq:{tag}
```

## ToDo:
- integration-pages docker
- use maven plugin to build images with current code, version etc.
