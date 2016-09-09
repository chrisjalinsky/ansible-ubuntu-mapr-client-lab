# Mapr Streams lab environment
###Dependencies
* Ansible 2.0
* Vagrant 1.8.5
* Virtualbox 5.0.20 r106931

###Prerequisites
* This guide uses a default 5.2 Mapr cluster installation, therefore the following configuration details are already established endpoints and servers on the hosts ```mapr[1:3].lan``` (See the screenshot for service details.)
* The Mapr cluster name has been set to ```mapr.cluster.lan```
* CLDB Node: ```mapr2.lan:7222```
* History Server: ```mapr3.lan```
* Zookeeper Nodes: ```mapr1.lan:5181,mapr2.lan:5181,mapr3.lan:5181```
* DNS has been established within the network, resolving forward and reverse records for these hosts.
* Bridged setup. Although this could easily be done with static IPs.

###Vagrantfile
The ```vagrant up``` command will, by default, build 2 mapr-client hosts using the ansible/hosts.yaml API compatible file. Thereforw we use the dynamic inventory script to read the hosts.yaml file. Adjust as necessary.

###Ansible Playbook

* Set the variables for the Mapr cluster
* Pass in mapr-ecosystem install packages
* Pass in environment variables

Run the playbook from the project's root directory:
```
ansible-playbook provision_mapr_clients.yaml -i inventory.py
```

##mapr-kafka
###Mapr Streams
On a Mapr cluster node, complete the following.

####Create the Mapr streams:
```
maprcli stream create -path /sample-stream
maprcli stream edit -path /sample-stream -produceperm p -consumeperm p -topicperm p
```

####List the Stream topics
```
maprcli stream topic list -path /sample-stream
```

####Create the topics:
```
$ maprcli stream topic create -path /sample-stream  -topic fast-messages
$ maprcli stream topic create -path /sample-stream  -topic summary-markers
```

##Key Ansible commands

####Configure the Mapr client:
```
/opt/mapr/server/configure.sh -N mapr.cluster.lan -c -C mapr1.lan:7222 -HS mapr3.lan -Z mapr1.lan:5181,mapr2.lan:5181,mapr3.lan:5181
```

##Mapr Sample Apps:
The lab environment uses a sample Git repo as a basis. Check out this [https://github.com/mapr-demos/mapr-streams-sample-programs](Git Repo) for more information.

###Startup a Producer on mclient1:
```
java -cp `mapr classpath`:./mapr-streams-examples-1.0-SNAPSHOT-jar-with-dependencies.jar com.mapr.examples.Run producer
```

###Startup a consumer on mclient2:
```
java -cp `mapr classpath`:./mapr-streams-examples-1.0-SNAPSHOT-jar-with-dependencies.jar com.mapr.examples.Run consumer
```

##Additional Mapr useful commands

###Hadoop FS list
```
mapr@mclient1:~$ /opt/mapr/hadoop/hadoop-2.7.0/bin/hadoop fs -ls /
```

###Mapr Streams 
```
mapr@mclient1:~$ mapr streamanalyzer -path /apps/racing/stream -topics car4 -printMessages true
```

###Dbshell (saving streams to MaprDB, like the racing stream demo)
```
mapr@mapr1:~$ mapr dbshell

maprdb mapr:> find /apps/racing/stream
```

##mapr-oozie
###Oozie Installation details

The ansible mapr_client role follows this guide for ecosystem component installations [here](http://maprdocs.mapr.com/home/AdvancedInstallation/InstallOozie.html).

```apt-get install mapr-oozie``` which installs ```mapr-oozie-internal```

The playbook reconfigures mapr client:
```
/opt/mapr/server/configure.sh -R -N mapr.cluster.lan -c -C mapr1.lan:7222 -HS mapr3.lan -Z mapr1.lan:5181,mapr2.lan:5181,mapr3.lan:5181
```

Finally, the role templates the file:
```
/opt/mapr/hadoop/hadoop-2.7.0/etc/hadoop/core-site.xml
```

####NOTE:
The role leaves out the final ```maprcli``` commands, since the role doesn't connect to the cluster.

###Restart the MRv1 or YARN services
Use the Web UI or run the following from the Oozie server:
```
mapr@mapr1:~$ maprcli node services -name nodemanager -action restart -nodes mapr1.lan mapr2.lan mapr3.lan
mapr@mapr1:~$ maprcli node services -name resourcemanager -action restart -nodes mapr2.lan
```
Test connection:
```
mapr@mclient2:~$ /opt/mapr/oozie/oozie-4.2.0/bin/oozie admin -status
```