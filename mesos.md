# Playing around with Mesos on Vagrant

## TL;DR

1. Here's a repo: [VirtusLab/mesos-on-vagrant](https://github.com/VirtusLab/mesos-on-vagrant),
2. Clone it, make all your RAM free and setup VirtualBox,
3. `vagrant up` ,
4. Congratulations, you've 6-node (3 master, 3 slave nodes) "highly-available" cluster running on virtual machines.


## End notes - to begin with :-)

As it's the only part (despite of short descriptions of each service), I'll begin in unconventional way - with the sum up of gained experience.
Full installation procedure is covered by Ansible playbook (used to provision Vagrant VMs) in the repository mentioned above.
 
We had no physical machines to spare and wanted to know if Mesos suits our needs, so for test-purposes, I've formed 3-node cluster on virtual machines. In Vagrantfile slave and master roles has been separated. I've initially started with 2, as it is the most typical number of nodes in our deployments, then discovered an ancient truth:

``Shalt thou run one, three or more nodes. Two shalt thou not run. Two is right out. Once the two nodes, being the forbidden number, is run it makes no sense - it won't form a cluster, as it can't form a quorum.`` 

So in this case an additional ZK node will be required, serving the same role as "replication witness" in SQL Server.

Running Mesos services takes approx. 1GB of RAM on each node. Not that bad, but consider assigning at last 2.5GB for each VM to have enough spare memory for applications.

Furthermore, I was initially scarred of computation overhead for running this whole stack - unnecessarily, as with some application up and running, it just calmly idles in the background.

Furthermore, Marathon does cgroup isolation for tasks, so limiting CPU share, memory, network and other resources or capabilities is possible on per app basis.

Unfortunately, there's no easy and convenient way for running our database-of-choice - PostgreSQL - on Mesos, but currently-incubated Apache Cotton (formerly Mysos) is available for MySQL.

Still Mesos is a nice friend for Docker and running mixed workload (eg. batch processing jobs) along the same group machines.

### Worth to mention fully-fledged "batteries included" Mesos platforms
- [Mantl](https://github.com/CiscoCloud/microservices-infrastructure) (formerly Cisco Microservices Infrastructure)
- [Capgemini Apollo](https://github.com/Capgemini/Apollo)
- [Yelp PaaSTA](https://github.com/Yelp/paasta)
- [Mesosphere DCOS](https://mesosphere.com/product)

## Installing step-by-step

First of all, we need to get Mesos somehow. We were a bit short of time, so I headed for pre-compiled packages distributed by Mesosphere.

Their repository configuration for CentOS 7 can be obtained from `http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-3.noarch.rpm` .

### Deploying ZooKeeper
[ZooKeeper](https://zookeeper.apache.org/) is a distributed coordination system capable of maintaining configuration information and providing distributed synchronization. It's used in Mesos cluster for tracking list of Mesos master instances and election of the leader.

For precise commands required to perform all steps listed below and templates of configuration files, please see Ansible playbook mentioned in the first section.

1. Install `zookeeper` on all ZK nodes (if using Mesosphere repository, it's named `mesosphere-zookeeper`).
2. Configure list of ZK nodes in `/etc/zookeeper/conf/zoo.cfg` (server.1, server.2, ...)
3. Set ZK node ID corresponding to its IP address in `/var/lib/zookeeper/myid` file.
4. Enable and start `zookeeper` service.

Instead of configuring this part by hand, you can also consider running Exhibitor - a supervisor for ZooKeeper, which also provides web GUI for exploring stored data.

### Deploying Mesos-master
[Mesos](https://mesos.apache.org/) Master is an endpoint between you and Mesos cluster. It handles resource allocation for your frameworks, allocates resources on slaves and keeps track of task execution.

1. Install `mesos`
2. Configure list of ZK servers in `/etc/mesos/zk`, it should be formatted like `zk://first.ip.address:2181,second.ip.address:2181,third.ip.address:2181/mesos`
3. If multiple Ethernet adapters are present (or multiple IP addresses on a single one), set proper IP address in `/etc/mesos-master/ip`, otherwise it will use the first IPv4 address.
4. Enable and start `mesos-master` service.

I won't cover slave and frameworks authentication, as for this particular moment only file-based authentication is supported and it's not needed to give Mesos a try.

### Deploying Mesos-slave
Mesos Slave reports itself and available resources to the master, then runs provisioned tasks.

1. Install `mesos`
2. Configure, again, list of ZK servers in `/etc/mesos/zk`, it should be formatted like `zk://first.ip.address:2181,second.ip.address:2181,third.ip.address:2181/mesos`.
3. If multiple Ethernet adapters are present (or multiple IP addresses on/etc/mesos-slave/containerizers a single one), set proper IP address in `/etc/mesos-slave/ip`, otherwise it will use first address.
4. If you're willing to run Docker containers, install Docker and put "docker,mesos" into `/etc/mesos-slave/containerizers`.
4. Enable and start `mesos-slave` service.
5. Connect to your Mesos master via HTTP at port `:5050` and check if all of your slaves got registered.

In point 5., you can access any Mesos master server, as you will get automagically redirected to one, which is currently elected as a leader.
For details of how leader election works, see [this section of ZooKeeper manual](http://zookeeper.apache.org/doc/trunk/recipes.html#sc_leaderElection).

I'd suggest running local Docker registry in your network, otherwise you may want to make `executor_registration_timeout` higher than the default one (again, see Ansible playbook).

### Deploying Marathon

[Marathon](https://github.com/mesosphere/marathon) is an Mesos framework for running generic, long-running applications. It makes running anything - from Bash script up to your Docker-wrapped complicated Play app - easy, as no specific integration with Mesos inside them is needed. It makes starting, stopping, down- and up-scaling your job easy and accessible via REST API and eye-candy GUI.
It's Mesosphere-founded easily-deployable alternative to Apache Aurora.

1. Install `marathon`
2. It uses same configuration files for ZK servers list as Mesos
3. Enable and start `marathon` service.

Quoting [Bill Farner](https://twitter.com/wfarner): ["ultimately these systems are sophisticated machinery to execute shell code somewhere in a cluster"](https://stackoverflow.com/questions/28651922/marathon-vs-aurora-and-their-purposes), so let's do it.
 
At this point you should be able to run a simple task, like a shell one-liner.
Access the Marathon service via HTTP at port `:8080` and add new application, eg. an infinite Bash loop: `hostname; while :; do date; sleep 1; done` . It should start spinning somewhere in your Mesos "cluster", if any of the slaves has sufficient resources available (if it won't scale up even to 1 instance, it's probably that case). Then, you can access stdout and stderr of your tasks via Mesos web UI.

### Deploying Chronos

There's one more interesting service which can be easily deployed - a distributed and fault-tolerant scheduler named [Chronos](https://github.com/mesos/chronos). You can use it for making sure your periodic task will be executed without ensuring that your indestructible, dedicated Cron server has not failed nor implementing complicated lock system in Bash.

1. Install `chronos`
2. It uses same configuration files for ZK servers list as Mesos
3. Enable and start `chronos` service.

As it takes ISO8601 interval notation, running a task each 16 minutes is no longer a nightmare. I'm getting Chronos-addicted.

### Mesos-DNS

Another nice-to-have thing is [mesosphere/mesos-dns](https://github.com/mesosphere/mesos-dns). It makes services discovery really easy, in a manner similar to [Consul](https://www.consul.io/) - by providing DNS and REST API.
I'm a bit scared of running automagic scripts to install software, so I've downloaded, compiled and RPM-packaged it by myself this time. The result RPM can be found in the repository mentioned at the beginning of this post.

If you're willing to use your cluster as a hive for swarm of microservices, you'll probably need it. As it provides "A" records for every running Marathon app together with "SRV" with its auto-assigned port, it is a very handy tool to configure front end server or load balancer.

Take a peek on Mesosphere DCOS, as it has nice dynamic Nginx configuration you can inspire from. I'm not sure about performance of this approach.

As an alternative to making your own tooling, you can consider using marathon-lb from Mesosphere (it's HAProxy backed).

Or abandon everything you've done up to here, and just run a Mesosphere DCOS Community Edition cluster on Amazon AWS ;-)


## Missing things

### Shared storage

One more thing you'd set up is shared storage. I wasn't interested in setting up HDFS nor other distributed file system, so just used network attached storage available from my cluster.  To try it out with Spark driver for Mesos or self-contained applications, it's fully sufficient to have them available from just any HTTP server or S3.

## Monitoring

I'd consider exporting Mesos metrics into Graphite or Prometheus. Both seem to be widely-used solutions.



