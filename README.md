# mesosMarathonDebian
Apache Mesos and Mesophere Marathon setup on Debian Jessie

This guide sets up Apache Mesos and Mesophere Marathon on 2 machines.

# Contents

- Versions
- Hosts
- Setup
- Master setup
- Node setup
- Run a job

# Versions

(latest as of today)

- Debian Jessie
- Apache Mesos 0.25
- Mesophere Marathon 0.11.1
- docker 1.9

# Hosts

- 172.16.43.248 Master
- 172.16.43.183 Node

# Setup

## Master

Install java8, mesos, marathon and configure cluster.

```console
apt-key adv --keyserver keyserver.ubuntu.com --recv E56151BF
DISTRO=$(lsb_release -is | tr '[:upper:]' '[:lower:]')
CODENAME=$(lsb_release -cs)
echo "deb http://repos.mesosphere.com/${DISTRO} ${CODENAME} main" | tee /etc/apt/sources.list.d/mesosphere.list
echo "deb http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main" | tee /etc/apt/sources.list.d/webupd8team-java.list
echo "deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main" | tee -a /etc/apt/sources.list.d/webupd8team-java.list
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys EEA14886
apt-get -y update
apt-get install -y oracle-java8-installer mesos marathon

echo '1' > /etc/zookeeper/conf/myid
sed -i -e 's/#server.1=zookeeper1:2888:3888/server.1=172.16.43.248:2888:3888/g' /etc/zookeeper/conf/zoo.cfg
echo "zk://172.16.43.248:2181/mesos" > /etc/mesos/zk
echo 'IP=172.16.43.248' >> /etc/default/mesos-master
echo 'easy' > /etc/mesos-master/cluster
echo '172.16.43.248' > /etc/mesos-master/hostname
service zookeeper restart
service mesos-slave stop
update-rc.d -f mesos-slave remove
service mesos-master restart
service marathon restart
```

## Node

Install mesos, docker and configure node to connect to master.

```console
apt-key adv --keyserver keyserver.ubuntu.com --recv E56151BF
DISTRO=$(lsb_release -is | tr '[:upper:]' '[:lower:]')
CODENAME=$(lsb_release -cs)
echo "deb http://repos.mesosphere.com/${DISTRO} ${CODENAME} main" | tee /etc/apt/sources.list.d/mesosphere.list
apt-get -y update 
apt-get -y install mesos
service zookeeper stop
echo "zk://172.16.43.248:2181/mesos" > /etc/mesos/zk
echo 'IP=172.16.43.183' >> /etc/default/mesos-slave
service mesos-master stop
update-rc.d -f mesos-master remove
service mesos-slave restart


# setup docker
apt-get update -y ;\
apt-get install -y apt-transport-https ;\
apt-key adv --keyserver hkp://pgp.mit.edu:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D ;\
echo "deb https://apt.dockerproject.org/repo debian-jessie main" > /etc/apt/sources.list.d/docker.list ;\
apt-get update -y ;\
apt-get upgrade -y ;\
apt-get install -y docker-engine bridge-utils


echo 'docker,mesos' > /etc/mesos-slave/containerizers
echo '5mins' > /etc/mesos-slave/executor_registration_timeout
service mesos-slave restart
```

# Run a job

Create a script to send json configurations to Marathon.
```console
#!/bin/bash
curl -X POST -H "Content-Type: application/json" 172.16.43.248:8080/v2/apps/ -d@"$@"
```

Write a simple.json to start ubuntu in a container.
```json
simple.json
{
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "libmesos/ubuntu"
    }    
  },
  "id": "ubuntu",
  "instances": 1,
  "cpus": 0.25,
  "mem": 230,
  "uris": [],
  "cmd": "while sleep 10; do date -u +%T; done"
}
```

Start the container.
```console
./l.sh simple.json
```

Use 'mesos ps' to list the jobs in the cluster.
```console
export PYTHONPATH=/usr/lib/python2.7/site-packages/
MASTER=$(mesos-resolve `cat /etc/mesos/zk`)
mesos ps --master=$MASTER
```


