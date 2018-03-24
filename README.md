FORK NOTES
==========

in this fork I made some customizations to let the module work properly inside official Zabbix-Agent Ubuntu or CentOS docker image.

IMPORTANT: you need to compile module using provided [Compilation](#compilation) guidelines before use.

Refer to [Installation](#installation) for installation guidelines.

---

If you like or use this project, please provide feedback to author - Star it ★
and [write what's missing for you](https://docs.google.com/forms/d/e/1FAIpQLSdYIokAyIMs2Qv19fzPxMWBubS9ESOYjJ2w_P222k5SuQuvoA/viewform).

Monitoring of Docker container by using Zabbix. Available CPU, mem,
blkio, net container metrics and some containers config details, e.g. IP, name, ...
Zabbix Docker module has native support for Docker containers (Systemd included)
and should also support a few other container types (e.g. LXC) out of the box.
Please feel free to test and provide feedback/open issue.
The module is focused on performance, see section
[Module vs. UserParameter script](#module-vs-userparameter-script).

Please donate to the author, so he can continue to publish other awesome projects
for free:

[![Paypal donate button](http://jangaraj.com/img/github-donate-button02.png)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=8LB6J222WRUZ4)

# Installation

1. Import provided template [Zabbix-Template-App-Docker.xml](https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/template/Zabbix-Template-App-Docker.xml).
2. Configure your Zabbix agent(s) - load your [compiled](#compilation) `zabbix_module_docker.so` by putting this file into `.../zbx_env/var/lib/zabbix/modules`<br>
https://www.zabbix.com/documentation/3.0/manual/config/items/loadablemodules
3. mount these volumes in agent container (docker-compose example):
```YAML
  volumes:
   - /etc/localtime:/etc/localtime:ro
   - /etc/timezone:/etc/timezone:ro
   - /var/run:/host/var/run:ro # to use in module
   - /proc:/host/proc:ro # to use in module
   - /sys/fs/cgroup:/sys/fs/cgroup:ro # to use in module
   - ./zbx_env/etc/zabbix/zabbix_agentd.d:/etc/zabbix/zabbix_agentd.d:ro
   - ./zbx_env/var/lib/zabbix/modules:/var/lib/zabbix/modules:ro
   - ./zbx_env/var/lib/zabbix/enc:/var/lib/zabbix/enc:ro
   - ./zbx_env/var/lib/zabbix/ssh_keys:/var/lib/zabbix/ssh_keys:ro
```
4. add **zabbix** user to **docker** group with the same gid of docker group in host inside container using an entrypoint shell script then run default /usr/bin/docker-entrypoint.sh 

Available templates:

- [Zabbix-Template-App-Docker.xml](https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/template/Zabbix-Template-App-Docker.xml) - standard (recommended) template
- [Zabbix-Template-App-Docker-active.xml](https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/template/Zabbix-Template-App-Docker-active.xml) - standard template with active checks
- [Zabbix-Template-App-Docker-Mesos-Marathon-Chronos.xml](https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/template/Zabbix-Template-App-Docker-Mesos-Marathon-Chronos.xml) - template for monitoring of Docker containers in Mesos cluster (Marathon/Chronos)

You can use Docker image [monitoringartist/zabbix-templates](https://hub.docker.com/r/monitoringartist/zabbix-templates/) for import of Zabbix-Template-App-Docker.xml template. For example:

```bash
docker run --rm \
  -e XXL_apiurl=http://zabbix.org/zabbix \
  -e XXL_apiuser=Admin \
  -e XXL_apipass=zabbix \
  monitoringartist/zabbix-templates
```

# Grafana dashboard

Custom Grafana dashboard for Docker monitoring with used Zabbix Docker (Mesos, Marathon/Chronos) templates are available in [Grafana Zabbix dashboards repo](https://github.com/monitoringartist/grafana-zabbix-dashboards).

![Grafana dashboard Overview Docker](https://raw.githubusercontent.com/monitoringartist/grafana-zabbix-dashboards/master/overview-docker/overview-docker.png) 

# Available metrics

Note: cid - container ID, two options are available:

- full container ID (macro *{#FCONTAINERID}*), e.g.
*2599a1d88f75ea2de7283cbf469ea00f0e5d42aaace95f90ffff615c16e8fade*
- human name or short container ID (macros *{#HCONTAINERID}* or *{#SCONTAINERID}*) - prefix "/" must be used, e.g.
*/zabbix-server* or */2599a1d88f75*

| Key | Description |
| --- | ----------- |
| **docker.discovery[\<par1\>,\<par2\>,\<par3\>]** | **LLD container discovering:**<br>Only running containers are discovered.<br>[Additional Docker permissions](#additional-docker-permissions) are needed when you want to see container name (human name) in metrics/graphs instead of short container ID. Optional parameters are used for definition of HCONTAINERID - docker.inspect function will be used in this case.<br>For example:<br>*docker.discovery[Config,Env,MESOS_TASK_ID=]* is recommended for Mesos/Chronos/Marathon container monitoring<br>Note 1: *docker.discovery* is faster version of *docker.discovery[Name]*<br>Note 2: Available macros:<br>*{#FCONTAINERID}* - full container ID (64 character string)<br>*{#SCONTAINERID}* - short container ID (12 character string)<br>*{#HCONTAINERID}* - human name of container<br>*{#SYSTEM.HOSTNAME}* - system hostname |
| **docker.port.discovery[cid,\<protocol\>]** | **LLD published container port dicovering:**<br>**protocol** - port protocol, which should be discovered, default value *all*, available protocols: *tcp,udp* |
| **docker.mem[cid,mmetric]** | **Memory metrics:**<br>**mmetric** - any available memory metric in the pseudo-file memory.stat, e.g.: *cache, rss, mapped_file, pgpgin, pgpgout, swap, pgfault, pgmajfault, inactive_anon, active_anon, inactive_file, active_file, unevictable, hierarchical_memory_limit, hierarchical_memsw_limit, total_cache, total_rss, total_mapped_file, total_pgpgin, total_pgpgout, total_swap, total_pgfault, total_pgmajfault, total_inactive_anon, total_active_anon, total_inactive_file, total_active_file, total_unevictable*, Note: if you have a problem with memory metrics, be sure that memory cgroup subsystem is enabled - kernel parameter: *cgroup_enable=memory* |
| **docker.cpu[cid,cmetric]** | **CPU metrics:**<br>**cmetric** - any available CPU metric in the pseudo-file cpuacct.stat/cpu.stat, e.g.: *system, user, total (current sum of system/user* or container [throttling metrics](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-cpu.html): *nr_throttled, throttled_time*<br>Note: CPU user/system/total metrics must be recalculated to % utilization value by Zabbix - *Delta (speed per second)*. |
| **docker.dev[cid,bfile,bmetric]** | **Blk IO metrics:**<br>**bfile** - container blkio pseudo-file, e.g.: *blkio.io_merged, blkio.io_queued, blkio.io_service_bytes, blkio.io_serviced, blkio.io_service_time, blkio.io_wait_time, blkio.sectors, blkio.time, blkio.avg_queue_size, blkio.idle_time, blkio.dequeue, ...*<br>**bmetric** - any available blkio metric in selected pseudo-file, e.g.: *Total*. Option for selected block device only is also available e.g. *'8:0 Sync'* (quotes must be used in key parameter in this case)<br>Note: Some pseudo blkio files are available only if kernel config *CONFIG_DEBUG_BLK_CGROUP=y*, see recommended docs. |
| **docker.inspect[cid,par1,\<par2\>,\<par3\>]** | **Docker inspection:**<br>Requested value from Docker inspect JSON object (e.g. [API v1.21](http://docs.docker.com/engine/reference/api/docker_remote_api_v1.21/#inspect-a-container)) is returned.<br>**par1** - name of 1st level JSON property<br>**par2** - optional name of 2nd level JSON property<br>**par3** - optional name of 3rd level JSON property or selector of item in the JSON array<br>For example:<br>*docker.inspect[cid,Config,Image], docker.inspect[cid,NetworkSettings,IPAddress], docker.inspect[cid,Config,Env,MESOS_TASK_ID=], docker.inspect[cid,State,StartedAt], docker.inspect[cid,Name]*<br>Note 1: Requested value must be plain text/numeric value. JSON objects and booleans are not supported.<br>Note 2: [Additional Docker permissions](#additional-docker-permissions) are needed.<br>Note 3: If you use selector for selecting value in array, then selector string is removed from returned value. |
| **docker.info[info]** | **Docker information:**<br>Requested value from Docker info JSON object (e.g. [API v1.21](http://docs.docker.com/engine/reference/api/docker_remote_api_v1.21/#display-system-wide-information)) is returned.<br>**info** - name of requested information, e.g. *Containers, Images, NCPU, ...*<br>Note: [Additional Docker permissions](#additional-docker-permissions) are needed. |
| **docker.stats[cid,par1,\<par2\>,\<par3\>]** | **Docker container resource usage statistics:**<br>Docker version 1.5+ is required<br>Requested value from Docker stats JSON object (e.g. [API v1.21](http://docs.docker.com/engine/reference/api/docker_remote_api_v1.21/#get-container-stats-based-on-resource-usage)) is returned.<br>**par1** - name of 1st level JSON property<br>**par2** - optional name of 2nd level JSON property<br>**par3** - optional name of 3rd level JSON property<br>For example:<br>*docker.stats[cid,memory_stats,usage], docker.stats[cid,network,rx_bytes], docker.stats[cid,cpu_stats,cpu_usage,total_usage]*<br>Note 1: Requested value must be plain text/numeric value. JSON objects/arrays are not supported.<br>Note 2: [Additional Docker permissions](#additional-docker-permissions) are needed.<br>Note 3: The most accurate way to get Docker container stats, but it's also the slowest (0.3-0.7s), because data are readed from on demand container stats stream. |
| **docker.cstatus[status]** | **Count of Docker containers in defined status:**<br>**status** - container status, available statuses:<br>*All* - count of all containers<br>*Up* - count of running containers (Paused included)<br>*Exited* - count of exited containers<br>*Crashed* - count of crashed containers (exit code != 0)<br>*Paused* - count of paused containers<br>Note: [Additional Docker permissions](#additional-docker-permissions) are needed.|
| **docker.istatus[status]** | **Count of Docker images in defined status:**<br>**status** - image status, available statuses:<br>*All* - all images<br>*Dangling* - count of dangling images<br>Note: [Additional Docker permissions](#additional-docker-permissions) are needed.|
| **docker.vstatus[status]** | **Count of Docker volumes in defined status:**<br>**status** - volume status, available statuses:<br>*All* - all volumes<br>*Dangling* - count of dangling volumes<br>Note 1: [Additional Docker permissions](#additional-docker-permissions) are needed.<br>Note2: Docker API v1.21+ is required|
| **docker.up[cid]** | **Running state check:**<br>1 if container is running, otherwise 0 |
| **docker.modver** | Version of the loaded docker module |
| | |
| **docker.xnet[cid,interface,nmetric]** | **Network metrics (experimental):**<br>**interface** - name of interface, e.g. eth0, if name is *all*, then sum of selected metric across all interfaces is returned (`lo` included)<br>**nmetric** - any available network metric name from output of command netstat -i:<br>*MTU, Met, RX-OK, RX-ERR, RX-DRP, RX-OVR, TX-OK, TX-ERR, TX-DRP, TX-OVR*<br>For example:<br>*docker.xnet[cid,eth0,TX-OK]<br>docker.xnet[cid,all,RX-ERR]*<br>Note 1: [Root permissions (AllowRoot=1)](#additional-docker-permissions) are required, because net namespaces (`/var/run/netns/`) are created/used <br>Note 2: **netstat** is needed to be installed and available in PATH|

Container log monitoring
========================

[Standard Zabbix log monitoring](https://www.zabbix.com/documentation/3.0/manual/config/items/itemtypes/log_items)
can be used. Keep in mind, that Zabbix agent must support active mode for log
monitoring. Stdout/stderr Docker container console output is logged by Docker
into file */var/lib/docker/containers/\<fid\>/\<fid\>-json.log* (fid - full container
ID = macro *{#FCONTAINERID}*). If the application in container is not able to
log to stdout/stderr, link log file to stdout/stderr. For example:

```bash
ln -sf /dev/stdout /var/log/nginx/access.log
ln -sf /dev/stderr /var/log/nginx/error.log
```

Example of *<fid>-json* log file:

```
{"log":"2015-07-03 00:15:05,870 DEBG fd 13 closed, stopped monitoring \u003cPOutputDispatcher at 37974528 for \u003cSubprocess at 37493936 with name php-fpm in state STARTING\u003e (stdout)\u003e\n","stream":"stdout","time":"2015-07-03T00:15:05.871956756Z"}
{"log":"2015-07-03 00:15:05,873 DEBG fd 17 closed, stopped monitoring \u003cPOutputDispatcher at 37974240 for \u003cSubprocess at 37493936 with name php-fpm in state STARTING\u003e (stderr)\u003e\n","stream":"stdout","time":"2015-07-03T00:15:05.875886957Z"}
{"log":"2015-07-03 00:15:06,878 INFO success: nginx entered RUNNING state, process has stayed up for \u003e than 1 seconds (startsecs)\n","stream":"stdout","time":"2015-07-03T00:15:06.882435459Z"}
{"log":"2015-07-03 00:15:06,879 INFO success: nginx-reload entered RUNNING state, process has stayed up for \u003e than 1 seconds (startsecs)\n","stream":"stdout","time":"2015-07-03T00:15:06.882548486Z"}
```

Recommended Zabbix log key for this case:

```
log[/var/lib/docker/containers/<fid>/<fid>-json.log,"\"log\":\"(.*)\",\"stream",,,skip,\1]
```

You can utilize Zabbix LLD for automatic Docker container log monitoring. In this case it'll be:

```
log[/var/lib/docker/containers/{#FCONTAINERID}/{#FCONTAINERID}-json.log,"\"log\":\"(.*)\",\"stream",,,skip,\1]
```

Images
======

Docker container CPU graph in Zabbix:
![Docker container CPU graph in Zabbix](https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/doc/zabbix-docker-container-cpu-graph.png)
Docker container memory graph in Zabbix:
![Docker container memory graph in Zabbix](https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/doc/zabbix-docker-container-memory-graph.png)
Docker container state graph in Zabbix:
![Docker container state graph in Zabbix](https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/doc/zabbix-docker-container-state-graph.png)

Additional Docker permissions
=============================

You have two options, how to get additional Docker permissions:

- Add zabbix user to docker group (recommended option):

```bash
usermod -aG docker zabbix
```

**Or**

- Edit zabbix_agentd.conf and set AllowRoot (Zabbix agent with root
permissions):

```bash
AllowRoot=1
```

Note: If you use Docker from RHEL/Centos repositories, then you have to
use *AllowRoot=1* option.

SELinux
-------
If you are on a system that has `SELinux` in enforcing-mode (check with `getenforce`), you can make it work with this SELinux module. This module will persist reboots. Save it, then run:

```bash
wget https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/selinux/zabbix-docker.te
checkmodule -M -m -o zabbix-docker.mod zabbix-docker.te
semodule_package -o zabbix-docker.pp -m zabbix-docker.mod
semodule -i zabbix-docker.pp
```

Compilation
===========

You have to compile the module if provided binary doesn't work on your system.
Basic compilation steps (please use right Zabbix branch version):

```bash
# Required CentOS/RHEL apps:   yum install -y wget autoconf automake gcc svn pcre-devel
# Required Debian/Ubuntu apps: apt-get install -y wget autoconf automake gcc subversion make pkg-config libpcre3-dev
# Required Fedora apps:        dnf install -y wget autoconf automake gcc subversion make pcre-devel
# Required openSUSE apps:      zypper install -y wget autoconf automake gcc subversion make pkg-config pcre-devel
# Required Gentoo apps 1:      emerge net-misc/wget sys-devel/autoconf sys-devel/automake sys-devel/gcc
# Required Gentoo apps 2:      emerge dev-vcs/subversion sys-devel/make dev-util/pkgconfig dev-libs/libpcre
# Source, use your version:    svn export svn://svn.zabbix.com/tags/3.2.7 /usr/src/zabbix
cd /usr/src/zabbix
./bootstrap.sh
./configure --enable-agent
mkdir src/modules/zabbix_module_docker
cd src/modules/zabbix_module_docker
wget https://raw.githubusercontent.com/mohammadrafigh/zabbix-docker-monitoring/master/src/modules/zabbix_module_docker/zabbix_module_docker.c
wget https://raw.githubusercontent.com/mohammadrafigh/zabbix-docker-monitoring/master/src/modules/zabbix_module_docker/Makefile
make
```

The output will be the binary file (dynamically linked shared object library) `zabbix_module_docker.so`, which can be loaded by Zabbix agent.

You can also use Docker for compilation. Example of Dockerfiles, which have been prepared for module compilation - https://github.com/mohammadrafigh/zabbix-docker-monitoring/tree/master/dockerfiles

Troubleshooting
===============

Edit your zabbix_agentd.conf and set DebugLevel:

    DebugLevel=4

Module debugs messages will be available in standard zabbix_agentd.log.

Issues and feature requests
===========================

Please use Github issue tracker.



**for more info refer to main repository.**
