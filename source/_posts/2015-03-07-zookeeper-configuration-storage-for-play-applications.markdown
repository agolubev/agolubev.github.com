---
layout: post
title: "Zookeeper - configuration storage for Play applications"
date: 2015-03-07 17:33:02 -0500
comments: true
categories: blog
tags: [play, zookeeper]
---

Most examples you can find are describing "Zookeeper service coordination" use case: 
"Some service registers itself during start in Zookeeper. This way it let others know about his availability.Also this service updates 
information about his state to coordinate efforts with others."
Another use case, which rarely mentioned, is storing static settings in Zookeeper instead of numerous of property files.
Despite simplicity of this approach there are several gaps that need to be filled. 
Below I will show advantages and disadvantages of mentioned approach, present Play plugin to work with Zookeeper as 
well as end to end process of storing/supporting settings.
<!-- more -->

As you may know Zookeeper is a service for storing and maintaining configuration of distributed systems.
It provides hierarchy data model wich is perfect for describing complex data centers and organizational structures.
You can setup zookeeper on several hosts to support fast reads and reliability of the service. 
Nodes define namespaces (ie folders) and can have data attached to them as well as children (files). All nodes can have attached data in binary format. 

Applications are usually rely on set of property files that reflects configuration for some particular environment. 
Approach is Ok when it's few configurations that you need to support (i.e. development, production) and small amount of servers to update config files.
As soon as number of such configuration variables increases - quantity of supporting configurations rises exponentially. 
For example you have 2 versions of web application that you need to support, 2 data centers, 2 environments like dev and prod and you are supporting two clients (with single tenant architecture). In case you have single config file you'd need to support `2^4=16` configurations. If you have config in different files then only `2*4=8`.
But the problem with updating config on each host is still remains. Here we need some mechanism that allows us to:

*	store configuration in one place to be accessible from any host
*	support levels of configuration to have general settings and environment/site specific settings
*	still need ability to track configuration in version control
*	have some access restrictions to be able to hide access info for production services

First approach is to use some shared file system. Here we need some additional logic around loading order (for ability to override some settings). 
Second approach is to use Zookeeper as storage for configuration. Zookeeper provides us with all four features we are looking for, still have few minor disadvantages:

*	there is no bulk load ability in Zookeeper API
*	relatively small size for node values

### Storing configuration at Zookeeper - overlook
Let's think about what we need to setup storing configuration in ZK and be able to support it:

*	Defining structure of hierarchical configurations
*	Creating/changing configuration
	*	Using UI/CLI tools
	*	Import/export to Zookeeper
	*	Storing in version control system
*	Loading config on Play application startup

Lets quicky run through all these points to show tools and approaches we can use.

###  Structure of configuration

The whole tree structure will be very much depend on what services we are using and the whole infrastructure we setup. Still you can consider this list as what we should consider as folders(levels) in config structure.

* global - default settings
* data center - can specify integrations and specific configurations
* organization - will need in case you are supporting several clients
* service - web, message service and any other kings of service specific configurations
* version - need to support different platform/service versions for 24/7 availbilty
* environment - can be Dev/Stage/QA/Prod
* default - branch for versions, services, etc. default settings

### Changing configuration
There are several ways of dealing with data in Zookeeper. The most routine way is changing something via UI tool. There are [Eclipse Plugin][eclipse-plugin],
[Idea Plugin][idea-plugin] (did not work for me though) and standard tool.
The last one is in Zookeeper distributive and you can run it via 

```sh
cd <zk-folder>/contrib/ZooInspector
java -cp zookeeper-3.4.6-ZooInspector.jar:lib/jtoaster-1.0.4.jar:../../lib/log4j-1.2.16.jar:../../zookeeper-3.4.6.jar org.apache.zookeeper.inspector.ZooInspector
```
There is also a command line utility CLI available in zookeeper installation. You can find good description [here][cli].

The problem here is that there is no standard tool for import and export subtree of Zookeeper data. Well, they have [zktreeutil][zktreeutil] 
but it's only for Linux.

There is one tool [zookeeper-util][zktreeutil] which is on ruby but so it will work on Mac and Windows. This tool helps keeping config properties up to date in zookeeper and in version control by doing export and import. Also zookeeper-util uses text format so it simplifies moving existing properties files to zookepeer. Export/import format looks like:

```
/
/conf
/conf/dev
/conf/dev/web
/conf/dev/web/jdbc.host::jdbc:mysql://10.10.10.1:3306/
/conf/dev/web/jdbc.user::dev
/conf/dev/web/jdbc.password::password
```

### PlayInZoo - load configuration on play start

PlayInZoo is fairly small plugin that allows you to load configuration from different branches of Zookeeper on startup. You need only to point where zookeeper is and what branches to use for loading. Here is configuration:

```scala Global.scala
override def onLoadConfig(config: Configuration, path: File, 
        classloader: ClassLoader, mode: Mode): Configuration = {
    config ++ PlayInZoo.loadConfiguration(config)
}
```
and
```properties application.conf
playinzoo.hosts=127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002
playinzoo.paths="/conf/dev/default->/conf/dev/web"
```

In example above all properties will be loaded from `default` branch - then from `web` branch. 
In case `default` already has values for some properties that will be overridden by values from `web`. 

### References
* [PlayInZoo - play plugin to load config from zookeeper][playinzoo]
* [Managing Configuration of Distributed System with Apache ZooKeeper][manage-config]
* [How-to: Use Apache ZooKeeper to Build Distributed Apps (and Why)][cli]
* [zktreeutil util][zktreeutil]

Hope this helps with overall understanging of configuration management process with Zookeeper and Play.

[manage-config]:   http://sysgears.com/articles/managing-configuration-of-distributed-system-with-apache-zookeeper/
[cli]:  http://blog.cloudera.com/blog/2013/02/how-to-use-apache-zookeeper-to-build-distributed-apps-and-why/
[zktreeutil]:	 https://code.google.com/p/bigstreams/source/browse/trunk/zookeeper-rpms/zookeeper/src/main/resources/contrib/zktreeutil/README.txt
[playinzoo]:	 https://github.com/agolubev/playinzoo
[eclipse-plugin]:http://www.massedynamic.org/mediawiki/index.php?title=Eclipse_Plug-in_for_ZooKeeper
[idea-plugin]: https://plugins.jetbrains.com/plugin/7364?pr=phpStorm 

