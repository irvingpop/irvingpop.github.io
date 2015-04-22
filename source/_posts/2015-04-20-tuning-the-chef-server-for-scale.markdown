---
layout: post
title: "Tuning the Chef Server for Scale"
date: 2015-04-20 13:51:36 -0700
comments: true
categories: chef server
---

In Chef's Customer Engineering team we are frequently asked for advice on tuning the Chef Server for high-scale situations. Below is a summary of what we generally tell customers. Note that these tuning settings are specific to Chef Server 12, which is the recommended version for any customer who cares about the performance of their Chef server.

<!-- more -->

## General Advice

### Understand the OSS components that make up the Chef Server
A good way to think about the Chef server is as a collection of Microservices components underpinned by OSS software:

* Nginx (openresty)
* PostgreSQL
* Solr
* RabbitMQ
* Redis
* Chef
* Erlang/OTP
* Ruby
* The Linux Kernel
  * LVM
  * Storage subsystem
  * Network stack

It's important to understand the performance characteristics, monitoring and troubleshooting of these components.  Especially Postgres, Solr, RabbitMQ and Linux systems in general. It's worth noting that the Chef server core is Open Source, and all if its code can be examined [on Github](https://github.com/chef/chef-server)

Because these components are glued together using Chef, it's highly recommended that you familiarize yourself with the [cookbooks that configure the Chef server when you run `chef-server-ctl reconfigure`](https://github.com/chef/opscode-omnibus/tree/master/files/private-chef-cookbooks)

### Have good monitoring in place

We don't provide prescriptive monitoring guidance at this time, but here's our advice:

* Use existing Open source software (Sensu, Nagios, etc) to collect metrics and test the health of the OSS components.  This should be fairly straightforward to set up.
  * Use [pgBadger](http://dalibo.github.io/pgbadger/) for Postgres log analysis
  * Install the [RabbitMQ Management Plugin](https://www.rabbitmq.com/management.html) for detailed monitoring of RabbitMQ
* Configure your monitoring systems and load balancers to query the Health status endpoint of erchef (https://mychefserver/_status)
* Run a graphite server. erchef will send detailed statistics if you set the following in your `chef-server.rb` file:
```ruby
folsom_graphite['enable'] = true
folsom_graphite['host'] = 'graphite.mycompany.com'
folsom_graphite['port'] = 2003
```
* Use Splunk or Logstash to collect and analyze your Chef server logs.
  - You can collect and graph useful performance data by graphing `/var/log/opscode/opscode-erchef/requests.log.N`, `/var/log/opscode/oc_bifrost/requests.log.N` and `/var/log/opscode/opscode-reporting/requests.log.N`
  - Each request line will show (in ms) various performance counter. For example: `req_time=20; rdbms_time=2; rdbms_count=3; authz_time=5; authz_count=1;`

### Think about API requests per second rather than node counts
A very common measurement for the size of Chef servers/clusters is the number of nodes they serve. However, this number is not terribly useful because of other elements that can cause very wide variation. Namely:

* The interval and splay of Chef client runs
  * 1000 nodes every hour == 500 nodes every 30 minutes
  * Insufficient splay can cause a "stampede condition" on the Chef server. Splay should be equal to the interval in order to get maximum smoothness of request load.
* The number and complexity of search requests and databag fetches performed during each Chef run
* The number of cookbooks depended on for each Chef run. More cookbooks adds loading to the depsolver and also to the Bookshelf service which serves cookbooks
* The size of node data, which we've seen range from 32kb to 5MB (the default maximum is 1MB but can be increased). This adds load to the indexing service (opscode-expander) as well as to Solr

Although it's not perfect, we've found that a good rule of thumb for examining active Chef servers is the number of API requests per second aggregated across the entire cluster. We've found that clusters which sustained higher than 125 API RPS started to experience occasional errors.

### DRBD: Don't do it
In the field we've found that DRBD has a negative impact on performance and availability of Chef server clusters. Specifically:

* Because DRBD uses synchronous replication, a block is not considered "committed to disk" until it has been confirmed by both nodes in the cluster. This adds significant latency to each IOP.
* DRBD's bandwidth is limited by the network throughput between the nodes. Dedicated cross-over links are not possible in all scenarios (for example VMs) which leads to low and inconsistent throughput.
* DRBD resyncs can take a very long time and greatly impact performance while running.
* Although DRBD protects against hardware failure, it does a very poor job of protecting against many classes of software failure. For example, a corrupt database is replicated whole to the other node, so failing over will not correct the system.


### Beware LVM snapshots impact on performance
LVM is generally recommended for storing all Chef Server data (`/var/opt/opscode` in standalone/tier installs and `/var/opt/opscode/drbd/data` in HA installs) because it provides the ability to expand disks on the fly and create crash-consistent snapshots.

However it's important to know that as LVM snapshots increase in size it is very detrimental to performance:

* http://www.percona.com/blog/2013/07/09/lvm-read-performance-during-snapshots/
* http://www.percona.com/blog/2009/02/05/disaster-lvm-performance-in-snapshot-mode/

Therefore it is recommend that snapshots are used to create consistent backups, but are immediately deleted after they are no longer needed.

## Chef Server tuning tips

### Server sizing

**Chef Server frontends:**

  * Frontends run stateless services only (erchef, bifrost, reporting, manage) and can be scaled horizontally.
  * They are almost always CPU bound, and only suffer memory or disk pressure during fault scenarios (typically because of backend issues).
  * A good starting point for frontends is 4 CPU cores and 8 GB RAM. Disk on frontends does not matter.

**Chef server backends:**

  * Backends mix a number of disk, memory and CPU bound services (Postgres, Solr, RabbitMQ, Expander)
  * A good starting point for backends is 8 CPU cores and 32 GB of RAM.
  * Flash-based storage is highly recommend, combined with the XFS filesystem and LVM.

### chef-server.rb tuning settings

**Database pooling:**

In the erlang OTP process model, the number of workers is limited by the size of the database connection pool (default 20). Increasing the database pool allows for more workers, but puts added memory pressure on the database service.

In order to handle the greater number of connections, you must also increase the Postgres `max_connections` value. This value must consider an erchef, bifrost and reporting process connecting from each frontend, plus an extra 20% for breathing room.

Suggested values for a high-performing cluster with 4-6 frontends:
```ruby
postgresql['max_connections'] = 1024
opscode_erchef['db_pool_size'] = 60
oc_bifrost['db_pool_size'] = 60
```

**Erchef to bifrost http connection pool:**
erchef also maintains a pool of http connections to bifrost, the authz service.  It's important to raise the initial and maximum number of connections with respect to the database pool sizes.

```ruby
oc_chef_authz['http_init_count'] = 100
oc_chef_authz['http_max_count'] = 200
```

**Erchef depsolver and keygen tuning:**
Two expensive computations that erchef must perform are the depsolver (a Ruby process which solves the cookbook dependencies) as well as the client key generator (which can be hit hard when large fleets of chef nodes are provisioned)

Suggested values:
```ruby
opscode_erchef['depsolver_worker_count'] = 4 # should equal the number of CPU cores
opscode_erchef['depsolver_timeout'] = 120000
opscode_erchef['keygen_cache_size'] = 1000
```

**Nginx cookbook caching:**
A new feature in Chef Server 12.0.4 is [Nginx cookbook caching](https://www.chef.io/blog/2015/02/18/cookbook-caching/). This takes load off of the backend Bookshelf service by storing cookbook files in Nginx.

Suggested values:
```ruby
opscode_erchef['nginx_bookshelf_caching'] = ":on"
opscode_erchef['s3_url_expiry_window_size'] = "100%"
```

**PostgreSQL tuning:**
We already tune PostgreSQL memory settings to sane values based on the backend's phyiscal RAM. For example, `effective_cache_size` is set to 50% of RAM, and `shared_buffers` to 25% of physical RAM.

To handle the heavy write load on large clusters, it is recommended to tune the checkpointer.

Suggested values:
```ruby
postgresql['checkpoint_segments'] = 100
postgresql['checkpoint_completion_target'] = 0.9
```

**Solr JVM tuning:**
By default we compute Solr's JVM heap size to be either 25% of system memory or 1024MB, whichever is smaller. Large chef server clusters should increase this value to smaller of 25% of system memory or 4096MB.

Suggested values:
```ruby
opscode_solr4['heap_size'] = 4096
opscode_solr4['new_size'] = 256
```

*WARNING: It is not recommended to use a JVM heap_size above 8GB, at that level the cost of Garbage Collection becomes too high and impacts performance worse than using a smaller heap*

