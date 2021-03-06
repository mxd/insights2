<img src="images/nuodb.svg" width="200" height="200" /> 

# NuoDB Insights - Visual Monitoring

[![Build Status](https://travis-ci.com/nuodb/nuodb-insights.svg?token=nYo6yHzhBM9syBKXYk7y&branch=master)](https://travis-ci.com/nuodb/nuodb-insights)

## Introduction

### Repository Structure:

| Directory | Description                                            |
|-----------|--------------------------------------------------------|
| conf      | dashboards and data sources for provisioning in grafana |
| deploy    | YAML configuration files for the monitor stack |
| images    | contains png included in this README |
| stable    | Helm Charts for Kubernetes Environments |

## Dashboards

### Dashboards Configuration

`conf/provisioning/dashboards/nuodb.yaml` defines the location of the dashboards.

## Quickstart with Docker compose

For a complete example on how to set up the NuoDB domain with NuoDB Insights, you can use `docker compose`.
This repository contains a Docker Compose file (`deploy/docker-compose.yml`) which will start:

- 1 Admin Processes
- 1 Storage Manager
- 2 Transaction Engines
- 2 NuoDB Collector containers (1 for SM, 1 for TE)
- 1 InfluxDB database
- 1 Grafana and NuoDB Dashboards
- 1 YCSB Demo Workload

Clone the NuoDB Insights repository and `cd` into it:

```
git clone https://github.com/nuodb/nuodb-insights.git
cd nuodb-insights
```

Then run `docker-compose up` to start the processes specified in the Docker Compose file:
```
docker-compose -f deploy/docker-compose.yaml up -d
```

Stop processes started with `docker-compose up` by running the following command:

```
docker-compose -f deploy/docker-compose.yaml down
```

If you already have a NuoDB domain running, and you only need NuoDB insights, you can use the `deploy/monitor-stack.yaml` file instead.

## Setup manually in Docker

### Download the Docker Images

```
docker pull nuodb/nuodb-ce:latest
docker pull nuodb/nuodb-collector:latest
docker pull influxdb:latest
docker pull grafana/grafana:latest
```

### Get NuoDB Insights
```
git clone https://github.com/nuodb/nuodb-insights.git
cd nuodb-insights
```

### Starting NuoDB Insights
First, you should create a Docker network.
All NuoDB components should be running on this network.
```
docker network create nuodb-net
```

Second, start an InfluxDB server.
We will use an init script that will generate the required databases.

```
docker run -d --name influxdb \
      --network nuodb-net \
      -p 8086:8086 \
      -p 8082:8082 \
      -v $PWD/deploy/initdb.sh:/docker-entrypoint-initdb.d/initdb.sh \
      influxdb:latest
```

As the final step, we will start Grafana with NuoDB dashboards.
```
docker run -d --name grafana \
      --network nuodb-net \
      -p 3000:3000 \
      --env INFLUX_HOST=influxdb \
      -v $PWD/conf/provisioning:/etc/grafana/provisioning \
      grafana/grafana:latest
```

You can now start your NuoDB domain with the NuoDB Collector.
For more info on how to start your domain, please refer to the [NuoDB Docker Blog Part I](https://nuodb.com/blog/deploy-nuodb-database-docker-containers-part-i) and the readme for [NuoDB Collector](https://github.com/nuodb/nuodb-collector).

You can use the example NuoDB YCSB workload generator to explore the various dashboards.
```
docker run -dit --name ycsb1 \
      --hostname ycsb1 \
      --network nuodb-net \
      --env PEER_ADDRESS=nuoadmin1 \
      --env DB_NAME=test \
      --env DB_USER=dba \
      --env DB_PASSWORD=goalie \
      nuodb/ycsb:latest /driver/startup.sh
```

## Setup in Kubernetes

### Helm Repository Structure

This GitHub repository contains the source for the packaged and versioned charts released in the [`gs://nuodb-charts` Google Storage bucket](https://console.cloud.google.com/storage/browser/nuodb-charts/) (the Chart Repository).

This Helm project currently contains only one Helm Chart called `insights`.
It contains all components required to install NuoDB Insights.
It is located in the [stable](stable/README.md) directory.

This Helm project does not currently contain any `incubator` Charts.

### Installation

If you are new to Kubernetes and Helm please read the [high-level description](stable/README.md) of this Helm repository.

If you are looking for specific configuration options see the [Insights Helm Chart Readme](stable/insights/README.md).

## Setup on Bare Metal

The following installation instructions apply to RedHat and CentOS distributions. For other platforms, see [InfluxDB](https://docs.influxdata.com/influxdb/latest/introduction/install/) and [Grafana](https://grafana.com/docs/grafana/latest/installation/) installation instructions.

### 1) Download and install `influxdb`

First install `influxdb` on some host (the same one referred to by `<hostinflux>` in the [NuoDB Collector](https://github.com/nuodb/nuodb-collector#configuration) installation instructions).

```
wget https://dl.influxdata.com/influxdb/releases/influxdb-1.8.3.x86_64.rpm
sudo yum localinstall -y influxdb-1.8.3.x86_64.rpm
sudo systemctl start influxdb
```

If not using `systemd`, `influxdb` can be started directly as follows:

```
env $(cat /etc/default/influxdb | xargs) influxd -config /etc/influxdb/influxdb.conf
```

### 2) Download and install `grafana` with NuoDB-Insights dashboards

On some host, install `grafana` and configure it to use the NuoDB-Insights dashboards.

```
wget https://dl.grafana.com/oss/release/grafana-7.3.1-1.x86_64.rpm
sudo yum localinstall -y grafana-7.3.1-1.x86_64.rpm

git clone https://github.com/nuodb/nuodb-insights.git
sudo rm -rf /etc/grafana/provisioning
sudo cp -r nuodb-insights/conf/provisioning /etc/grafana
sudo mkdir -p /etc/grafana/provisioning/notifiers /etc/grafana/provisioning/plugins
sudo chown -R root:grafana /etc/grafana/provisioning
sudo chmod 755 $(find /etc/grafana/provisioning -type d)
sudo chmod 640 $(find /etc/grafana/provisioning -type f)
```

Once installed, `grafana` can be started with `influxdb` as a datasource by using the `INFLUX_HOST` environment variable.

```
echo "INFLUX_HOST=<hostinflux>" >> /etc/sysconfig/grafana-server
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

If not using `systemd`, `grafana` can be started as follows:

```
sudo /etc/rc.d/init.d/grafana-server start
```

### 3) Install NuoDB Collector on all hosts with database processes

See the instructions for installing [NuoDB Collector](https://github.com/nuodb/nuodb-collector#setup-on-bare-metal) on bare-metal. NuoDB Collector should be set up on all hosts that have database processes.

Once all components have been set up, NuoDB performance can be visualized by navigating to `http://<hostgrafana>:3000` where `<hostgrafana>` is the host that the Grafana server was started on.

## Status of the Project

This project is still under active development, so you might run into [issues](https://github.com/nuodb/nuodb-insights/issues). If you do, please don't be shy about letting us know, or better yet, contribute a fix or feature.
