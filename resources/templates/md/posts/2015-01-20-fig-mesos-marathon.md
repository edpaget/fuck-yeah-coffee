{:title "Running Mesos and Marathon with Fig"
 :layout :post
 :tags ["mesos" "fig" "docker" "dev ops"]}

I recently read [gargar454's](https://medium.com/@gargar454) medium post [Deploy a Mesos cluster with 7 commands using docker](https://medium.com/@gargar454/deploy-a-mesos-cluster-with-7-commands-using-docker-57951e020586). 

## Background

### What's Mesos?

[Apache Mesos](http://mesos.apache.org) is a system for orchestrating cluter resources using containers. 

### What's Fig?

[Fig](http://fig.sh) allows you to declaritively run and link together docker containers. You can 

## Let's Do It

All right now to put the whole thing to together. The first thing Mesos needs is a Zookeeper quorum to talk to. Since a quorum can be a single Zookeeper and we're not worried about a loca development cluster's availabilty, we'll run a single Zookeeper node.

```
    zookeeper:
      image: edpaget/zookeeper
      command: -c localhost:2888:3888 -i 1
```

We're running my [Zookeeper image](https://github.com/edpaget/docker-zookeeper/blob/master/Dockerfile) which takes a few arguments. The `-c` option is a list of cluster members and `-i` is the Zookeeper node's id. You can run `docker run -i -t edpaget/zookeeper -h` to get full usage instructions.

And that's it for zookeeper. We don't need to mount any volumes or connect any other services ot it.

Next we'll run the Mesos Master node:

```
    mesosLeader:
      image: garland/mesosphere-docker-mesos-master:latest
      ports:
        - "5050:5050"
      environment:
        MESOS_ZK: zk://zookeeper:2181/mesos
        MESOS_PORT: 5050
        MESOS_QUORUM: 1
        MESOS_LOG_DIR: /var/log/mesos
        MESOS_REGISTRY: in_memory
        MESOS_WORK_DIR: /var/lib/mesos
      links:
        - zookeeper
```

There's a little bit more to unpack here, so let's go through it line-by-line.

+ `mesosLeader:` Alight maybe not quite line-by-line...
+ `image: garland/mesosphere-docker-mesos-master:latest` We're using [sekka1's Mesos images](https://github.com/sekka1/mesosphere-docker/blob/master/mesos-master/Dockerfile). Including a tag in the fig file prevents docker from pulling the entire repository.
+ `ports` Mesos exposes its admin interface on port 5050. We need to forward this to our localhost 5050 so we can open in it a web browser.
+ `environment` Mesos accepts options as either environment variables or through its commandline. 


The Mesos Slave configure follows similarly from the Master's:


```
    mesosFollower:
      image: garland/mesosphere-docker-mesos-master:latest
      entrypoint: mesos-slave
      environment:
        MESOS_LOGGING_LEVEL: INFO
        MESOS_LOG_DIR: /var/log/mesos
        MESOS_MASTER: zk://zookeeper:2181/mesos
      links:
        - zookeeper
```

The main difference is that we use the `entrypoint` key to define a new command that will be run by the docker container.

Finally Marathon:

```
    marathon:
      image: garland/mesosphere-docker-marathon:latest
      ports:
        - "8080:8080"
      command: --master zk://zookeeper:2181/mesos --zk zk://zookeeper:2181/mesos
        links:
        - zookeeper
```
