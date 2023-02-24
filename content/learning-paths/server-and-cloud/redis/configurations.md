---
# User change
title: "Redis deployment configurations"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Redis deployment configurations

##  Introduction to Redis
Redis, which stands for Remote Dictionary Server, is an open source in-memory data structure store used as a database, cache, message broker, and streaming engine. Redis has a variety of data types, including **bitmaps**, **hyperloglogs**, **geographic indexes**, **streams**, **lists**, **sets**, and **sorted sets with range queries**.

## Configuring Redis Server
We can configure Redis server using [redis.conf](https://redis.io/docs/management/config-file/) file. Alternatively, we can configure Redis server by [passing arguments via command line](https://redis.io/docs/management/config/#passing-arguments-via-the-command-line) when fewer configuration variables need to be set.

## Single node configuration
After installing Redis, by default it runs on localhost (`127.0.0.1`) at port `6379`. Hence, `127.0.0.1:6379` becomes unavailable for binding. Thus, we need to set the bind configuration option in the redis.conf file to `0.0.0.0`.

For a single node Redis server, we need to set:
- `--port` option providing the port number 
- `--daemonize` option set to `yes` to run redis in background.  

To connect to the remote Redis server, we need to use Redis Client (`redis-cli`) with:
- `-h` option providing hostname
- `-p` option providing the port number.  

## Multi-node configuration
A Redis multi-node cluster requires 3 primary and 3 replica nodes in minimal configuration to work properly.  

We can use 6 different ports of the same host as follows:
```console
redis-cli --cluster create HOST:port1 HOST:port2 HOST:port3 HOST:port4 HOST:port5 HOST:port6
```
This method has been used in official Redis [documentation](https://redis.io/docs/management/scaling/#create-and-use-a-redis-cluster) as well.

Alternatively, we can create 6 different hosts with Redis server running on same port as follows:
```console
redis-cli --cluster create HOST1:port HOST2:port HOST3:port HOST4:port HOST5:port HOST6:port
```
This method requires manual joining and configuration of each host and requires more resources. Hence, it is not a preferred method for multi-node cluster.


For creating a multi-node Redis cluster, we use the first approach, in which we need to create 6 folders having port number as name. For example, if we use ports `6001-6006`, we will have to create folders named `6001`, `6002` till `6006`. In each folder, we need to create a **redis.conf** file.

Here is the minimal template of **redis.conf** file
```console
protected-mode no
port {port_number}
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
daemonize yes
appendonly yes
```
**NOTE:-** The `{port_number}` will be same as folder name.  

To connect to the remote Redis multi-node cluster, we need to use Redis Client (`redis-cli`) with:
- `-c` option to enable cluster mode
- `-h` option providing hostname
- `-p` option providing the port number.
