---
title: Load Balancing with etcd and vulcand
component: load-balancing
author: Michael Mueller
pubdate: 2015-09-23 00:00:00
---

## Problem

If you want to make your distributed microservices accessible via HTTP so they can call each other or be accessed from the outside world vulcanproxy can be used.

---

## Overview

**Components:** [CoreOS](/tech/coreos/), [etcd](/tech/etcd/), [vulcand](/tech/vulcand/)

* CoreOS is a minimal Linux OS optimized to run containers
* [etcd](/tech/etcd/) is a clustered key value store used to store the configuration of vulcand
* vulcand is a progammable loadbalancer developed by [mailgun.com](https://www.mailgun.com/), an email service for devs


---

### Pros

- Interacts directly with etcd
- Configuration is distributed and fault tolerant stored
- Changes don't need a restart
- No config files needed

---

### Cons

- Still beta
- "Status: Under active development. Used at Mailgun on moderate workloads."

---

## How Vulcand works

Vulcand consists of three parts, frontends, backends and middlewares. The frontend is a URI path which can be matched using RegEx. This location is matched up with an backend, which is a set of servers to serve the request.
If the request matches a frontend, the traffic get routed to defined backend. Middlewares sit between frontend and backend and are able to change, intercept or reject requests.

---

### Frontends

[vulcand/frontends](https://docs.vulcand.io/proxy.html#frontends)

A frontend defines how requests should be routed to backends. Their definitions are composed of the following components. An example route definition will look like `Path("/foo/bar")`, which will match match the given path for all hosts. If you like to match only to a given Host the expression will look like `Host("example.com") && Path ("/foo/bar")`

```bash
$ etcdctl set /vulcand/frontends/example/frontend '{"Type": "http", "BackendId": "v1", "Route": "Host(`example.com`) && Path(`/`)"}'
```

#### Settings

In the frontend different controls are available

```json
{
  "Limits": {
    "MaxMemBodyBytes":<VALUE>, // Maximum request body size to keep in memory before buffering to disk
    "MaxBodyBytes":<VALUE>, // Maximum request body size to allow for this frontend
  },
  "FailoverPredicate":"IsNetworkError() && Attempts() <= 1", // Predicate that defines when requests are allowed to failover
  "Hostname": "host1", // Host to set in forwarding headers
  "TrustForwardHeader":<true|false>, // Time provider (useful for testing purposes)
}
```

---

### Backends

[vulcand/backends](https://docs.vulcand.io/proxy.html#backends-and-servers)

Vulcand load-balances requests within the backend and keeps the connection pool to every server. Frontends using the same backend will share the connections. Changes to the backend configuration can be done at any time and will triger a graceful reload of the settings.

```json
{
  "Timeouts": {
     "Read":"1s", // Socket read timeout (before we receive the first reply header)
     "Dial":"2s", // Socket connect timeout
     "TLSHandshake": "3s", // TLS handshake timeout
  },
  "KeepAlive": {
     "Period":"4s", // Keepalive period for idle connections
     "MaxIdleConnsPerHost":3, // How many idle connections will be kept per host
  }
}

```

---

## Implementation steps

This example is based on the coreos/example [coreos.com/blog/zero-downtime-frontend-deploys-vulcand/](https://coreos.com/blog/zero-downtime-frontend-deploys-vulcand/) running on a 3 node coreos-cluster deployed via Vagrant.

Example `Vagrantfile`, `user-data` and `config.rb` can be found here:
[https://github.com/muemich/coreos-vagrant-vulcand](https://github.com/muemich/coreos-vagrant-vulcand)

---

### 1. Set up base infrastructure

Add `172.17.8.101 example.com` to `/etc/hosts` on your host machine,


Launch one or many CoreOS machines and log in. For this example one is enough.

```bash
$ vagrant up
$ vagrant ssh core-01 -- -A
```
---

### 2. Ensure etcd and vulcand are running

```bash
core@core-01 ~ $ systemctl status etcd2
● etcd2.service - etcd2
   Loaded: loaded (/usr/lib64/systemd/system/etcd2.service; disabled; vendor preset: disabled)
  Drop-In: /run/systemd/system/etcd2.service.d
           └─20-cloudinit.conf
   Active: active (running) since Tue 2015-09-22 15:00:57 UTC; 54min ago
 Main PID: 967 (etcd2)
   Memory: 38.2M
      CPU: 34.202s
   CGroup: /system.slice/etcd2.service
           └─967 /usr/bin/etcd2

Sep 22 15:01:43 core-01 etcd2[967]: 2015/09/22 15:01:43 raft: aeb20866b279648e received vote from aeb20866b279648e at term 2
Sep 22 15:01:43 core-01 etcd2[967]: 2015/09/22 15:01:43 raft: aeb20866b279648e [logterm: 1, index: 3] sent vote request to 56da6d1265bdc9ed at term 2
Sep 22 15:01:43 core-01 etcd2[967]: 2015/09/22 15:01:43 raft: aeb20866b279648e [logterm: 1, index: 3] sent vote request to eefc97cb642769af at term 2
Sep 22 15:01:43 core-01 etcd2[967]: 2015/09/22 15:01:43 raft: aeb20866b279648e received vote from 56da6d1265bdc9ed at term 2
Sep 22 15:01:43 core-01 etcd2[967]: 2015/09/22 15:01:43 raft: aeb20866b279648e [q:2] has received 2 votes and 0 vote rejections
Sep 22 15:01:43 core-01 etcd2[967]: 2015/09/22 15:01:43 raft: aeb20866b279648e became leader at term 2
Sep 22 15:01:43 core-01 etcd2[967]: 2015/09/22 15:01:43 raft: raft.node: aeb20866b279648e elected leader aeb20866b279648e at term 2
Sep 22 15:01:43 core-01 etcd2[967]: 2015/09/22 15:01:43 etcdserver: setting up the initial cluster version to 2.1.0
Sep 22 15:01:43 core-01 etcd2[967]: 2015/09/22 15:01:43 etcdserver: published {Name:d7251d6864f0497294358cf18f811017 ClientURLs:[http://172.17.8.101:2379]} to cluster eba7d2dbe11be795
Sep 22 15:01:43 core-01 etcd2[967]: 2015/09/22 15:01:43 etcdserver: set the initial cluster version to 2.1.0
```

```bash
systemctl status vulcand
● vulcand.service - Vulcand
   Loaded: loaded (/etc/systemd/system/vulcand.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2015-09-22 15:02:49 UTC; 53min ago
  Process: 1070 ExecStartPre=/usr/bin/docker pull mailgun/vulcand:v0.8.0-beta.3 (code=exited, status=0/SUCCESS)
  Process: 1063 ExecStartPre=/usr/bin/docker rm vulcand (code=exited, status=1/FAILURE)
  Process: 979 ExecStartPre=/usr/bin/docker kill vulcand (code=exited, status=1/FAILURE)
 Main PID: 1331 (docker)
   Memory: 12.5M
      CPU: 123ms
   CGroup: /system.slice/vulcand.service
           └─1331 /usr/bin/docker run --name vulcand -p 80:80 -p 443:443 -p 8182:8182 -p 8181:8181 mailgun/vulcand:v0.8.0-beta.2 /go/bin/vulcand -apiInterface=0.0.0.0 -interface=0.0.0.0 -etcd=http://<IP:4001>

Sep 22 15:03:00 core-01 docker[1331]: 5c5ef0bea32a: Download complete
Sep 22 15:03:00 core-01 docker[1331]: 6c24554e26e5: Pulling metadata
Sep 22 15:03:01 core-01 docker[1331]: 6c24554e26e5: Pulling fs layer
Sep 22 15:03:02 core-01 docker[1331]: 6c24554e26e5: Download complete
Sep 22 15:03:02 core-01 docker[1331]: e7e8f43c66ae: Pulling metadata
Sep 22 15:03:03 core-01 docker[1331]: e7e8f43c66ae: Pulling fs layer
Sep 22 15:03:05 core-01 docker[1331]: e7e8f43c66ae: Download complete
Sep 22 15:03:05 core-01 docker[1331]: e7e8f43c66ae: Download complete
Sep 22 15:03:05 core-01 docker[1331]: Status: Downloaded newer image for mailgun/vulcand:v0.8.0-beta.2
Sep 22 15:03:05 core-01 docker[1331]: Sep 22 15:03:05.587: WARN PID:1 [supervisor.go:349] No frontends found
```

As there are no frontends deployed yet, the warning can be ignored.

---

### 3. Deploy test backend containers

Run web application __v1__ containers,

```bash
$ docker run -d --name example-v1.1 -p 8086:80 coreos/example:1.0.0
$ docker run -d --name example-v1.2 -p 8087:80 coreos/example:1.0.0
```

Configure Vulcand to proxy to __v1__ container,

---

### 4. Register backend containers

```bash
$ etcdctl set /vulcand/backends/v1/backend '{"Type": "http"}'
$ etcdctl set /vulcand/backends/v1/servers/v1.1 '{"URL": "http://172.17.8.101:8086"}'
$ etcdctl set /vulcand/backends/v1/servers/v1.2 '{"URL": "http://172.17.8.101:8087"}'
```

*To proxy to v1 containers:*

```
$ etcdctl set /vulcand/frontends/example/frontend '{"Type": "http", "BackendId": "v1", "Route": "Host(`example.com`) && Path(`/`)"}'
```

Then access to `example.com` and you can see the current version _1.0.0_ .

![example_com](https://cloud.githubusercontent.com/assets/680124/9721329/a21893b0-55d3-11e5-88de-1b0c45394076.png)

---

### Future work

Setup middlewares, which can be used to change, intercept or reject requests. Vulcand provides a cli-tool called `vulcanbundle` which will write a new main.go that imports the original [vulcand](https://github.com/mailgun/vulcand)  as a library and all extension supplied as parameters.

To make the registration process of new backends automatic, entries for each backend need to be created in etcd. This can be accomplished by a script that runs after a new backend is started, or by hooking into lifecycle events of schedulers.

---