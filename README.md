# docker-sling-hosting
This is a prototype Docker hosting of Sling instances.

For now, it connects a number of Sling instances to a common MongoDB database, and automatically makes
them available via a front-end HAProxy server. Logs are collected in a Graylog instance. All these services
run in Docker containers.

The goals are to experiment with Sling in Docker and demonstrate automatic reconfiguration of related services
like HAProxy when Sling instances appear or disappear.

Apart from some glue scripts to setup the containers there's little or no code here, the goal is to use off-the-shelf
tools as much as possible, to concentrate on the proof of concept aspects. 

This is obviously not production-ready.

## Prerequisites
You need a Docker server and `docker-compose` setup to run this.

Unless you already have such a setup, https://www.docker.com/docker-toolbox is a good way to get started.

You Docker host needs at least 4G of memory to run this cluster comfortably. See the `docker-machine` docs
for the relevant options for your machine backend, if applicable.

After installing `docker-compose`, you can test it from this folder, as follows:

	$ docker-compose -f docker-compose-test.yml up
	
	Starting dockerslinghosting_docker-setup-test_1...
	Attaching to dockerslinghosting_docker-setup-test_1
	docker-setup-test_1 | Congratulations, your docker-compose setup works. See the docker subfolder for the actual Sling hosting setup.
	dockerslinghosting_docker-setup-test_1 exited with code 0
	Gracefully stopping... (press Ctrl+C again to force)
   
If you see the _Congratulations, ..._ message it means your `docker-compose` setup works, and
this prototype should work as well.

## Sling Launchpad jar
Before building the Sling Docker images you'll need to copy a Sling Launchpad jar at `docker/sling/org.apache.sling.launchpad.jar`. You can find one at http://sling.apache.org/downloads.cgi ("Sling Standalone Application") if you don't want to build it yourself.

## /etc/hosts setup
To access the virtual hosts that this prototype sets up, you'll need entries like this in your /etc/hosts:

    192.168.99.100 alpha.example.com
    192.168.99.100 bravo.example.com
    192.168.99.100 charlie.example.com
    ...

matching the `SLING_DOMAIN` values defined in the `docker/docker-compose.yml` file.

Replace 192.168.99.100 with the address of your Docker host if needed. `docker-machine ip default` provides
that value if you are using `docker-machine`.

## Starting the cluster
To start the cluster, for now it's safer to start the `mongo`, `graylog` and `etcd` containers first, as (pending more
testing) there might be startup timing issues otherwise.

So, from the `docker` folder found under this `README` file:


    # build the Docker images - might Download the Web (tm) the first time it runs
	docker-compose build

    # start the infrastructure containers	
	docker-compose up -d mongo etcd graylog
	
	# wait a few seconds for those to start up, and start the other containers
	docker-compose up -d

After a few seconds, tests hosts like http://alpha.example.com should be proxied to the Sling container instances.

The HAProxy stats are available at http://alpha.example.com:81

## Configuring graylog
Aggregated logs are provided by graylog at http://alpha.example.com:9000 . The initial credentials are _admin/admin_.

To collect them you need to configure an input at http://alpha.example.com:9000/system/inputs - create an input of 
type GELF TCP on port 12201.

Once that's done the search interface at http://alpha.example.com:9000/ should show messages emitted by the Sling
containers at regular intervals. For now these are simulated, like "Hello from delta.example.com/a96333b3f7b7...", 
we'll need to connect the Sling logging subsystem to graylog using a specific Logback GELF appender.

## Adding more Sling hosts
Copying and adapting the `sling_001` section of the `docker/docker-compose.yml` file and running `docker-compose up SSS` where
SSS is the name of the new container should start a new Sling instance.

Make sure to give a unique port number and `SLING_DB` to each Sling instance if you do that.

Starting a new instance should cause it to be registered automatically in the HAproxy container, so the corresponding host (as
defined by the `SLING_DOMAIN` variable in the `docker-compose.yml`) should become available, provided you have added the 
corresponding `/etc/hosts` entry.

The logs of the new instance will also appear automatically in Graylog.

## TODO - known issues
The internal Sling instance port numbers should be assigned dynamically. And we shouldn't need to expose them outside of
the Docker host, this is done for debugging purposes for now.

No real logs are sent to graylog yet, the current setup just demonstrates how container logs can
be aggregated in graylog.

The cluster probably only works on a single Docker host so far, we'll need ambassador containers (or something like Mesos or Kubernetes) to make it multi-host.

## Tips & tricks (aka notes to self)
This rebuilds and runs a single container (here `haproxy`) in interactive mode, for debugging:

    docker run -p 80:80 -p 81:81 -it --link docker_etcd_1:etcd $(docker-compose build haproxy | grep "Successfully" | cut -d' ' -f3) bash
