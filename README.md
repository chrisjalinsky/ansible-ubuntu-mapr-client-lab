# Mapr Streams lab environment
###Dependencies
* Ansible 2.0
* Vagrant 1.8.5
* Virtualbox 5.0.20 r106931

###Prerequisites
* This guide uses a default [5.2 Mapr cluster installation](http://maprdocs.mapr.com/home/AdvancedInstallation/c_get_started_install.html), therefore the following configuration details are already established endpoints on the mapr cluster hosts ```mapr[1:3].lan``` (See the screenshot for service details.)
* The Mapr cluster name has been set to ```mapr.cluster.lan```
* CLDB Node: ```mapr2.lan:7222```
* History Server: ```mapr3.lan```
* Zookeeper Nodes: ```mapr1.lan:5181,mapr2.lan:5181,mapr3.lan:5181```
* DNS has been established within the network, resolving forward and reverse records for these hosts.
* Bridged setup. Although this could easily be done with static IPs.


####Note: This repo is heavily inspired by the following [Mapr streaming data from racing cars demo](https://github.com/mapr-demos/racing-time-series)
Spark libraries have been commented out because the download was taking so long for me. Need to clean up the installation plays

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
On a Mapr cluster node, complete the following stream and topic creation.

####Create the Mapr streams:
```
maprcli stream create -path /sample-stream
```
####Mapr introduces permissions into streams, therefore open permissions for writing
```
maprcli stream edit -path /sample-stream -produceperm p -consumeperm p -topicperm p
```

####Create the topics:
```
$ maprcli stream topic create -path /sample-stream  -topic fast-messages
$ maprcli stream topic create -path /sample-stream  -topic summary-markers
```

####List the Stream topics
```
maprcli stream topic list -path /sample-stream
```

####Configure the Mapr client:
```
/opt/mapr/server/configure.sh -N mapr.cluster.lan -c -C mapr1.lan:7222 -HS mapr3.lan -Z mapr1.lan:5181,mapr2.lan:5181,mapr3.lan:5181
```

##Mapr Sample Apps:
The lab environment uses a sample Git repo as a basis. Follow this [Git repo](https://github.com/mapr-demos/mapr-streams-sample-programs) for more information.
The mapr_client ansible role installs Java8 and Maven to compile the examples.

###Compile the examples on the mclients with Maven:
```
mapr@mclient1:/opt/mapr-streams-sample-programs$ mvn package
mapr@mclient2:/opt/mapr-streams-sample-programs$ mvn package
```

###Startup a producer on mclient1:
```
cd target
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

Run Example oozie yarn jobs:
```
mapr@mclient1:/opt/mapr/oozie/oozie-4.2.0$ tar xvfz ./oozie-examples.tar.gz -C /opt/mapr/oozie/oozie-4.2.0/
mapr@mclient1:/opt/mapr/oozie/oozie-4.2.0$ /opt/mapr/hadoop/hadoop-2.7.0/bin/hadoop fs -put examples maprfs:///user/mapr/examples
mapr@mclient1:/opt/mapr/oozie/oozie-4.2.0$ /opt/mapr/hadoop/hadoop-2.7.0/bin/hadoop fs -put examples/input-data maprfs:///user/mapr/input-data
mapr@mclient1:/opt/mapr/oozie/oozie-4.2.0$ /opt/mapr/hadoop/hadoop-2.7.0/bin/hadoop fs -chmod -R 777 maprfs:///user/mapr/examples
mapr@mclient1:/opt/mapr/oozie/oozie-4.2.0$ /opt/mapr/oozie/oozie-4.2.0/bin/oozie job -oozie="http://mapr3.lan:11000/oozie" -config /opt/mapr/oozie/oozie-4.2.0/examples/apps/map-reduce/job.properties -run

mapr@mclient1:/opt/mapr/oozie/oozie-4.2.0$ yarn application -list
16/09/09 06:27:44 INFO client.MapRZKBasedRMFailoverProxyProvider: Updated RM address to mapr2.lan/10.0.0.129:8032
Total number of applications (application-types: [] and states: [SUBMITTED, ACCEPTED, RUNNING]):1
                Application-Id	    Application-Name	    Application-Type	      User	     Queue	             State	       Final-State	       Progress	                       Tracking-URL
application_1473389464865_0003	oozie:action:T=map-reduce:W=map-reduce-wf:A=mr-node:ID=0000000-160828220950359-oozie-mapr-W	           MAPREDUCE	      mapr	 root.mapr	           RUNNING	         UNDEFINED	             5%	             http://mapr2.lan:36076


mapr@mclient1:/opt/mapr/oozie/oozie-4.2.0$ /opt/mapr/oozie/oozie-4.2.0/bin/oozie job -info 0000000-160828220950359-oozie-mapr-W
Job ID : 0000000-160828220950359-oozie-mapr-W
------------------------------------------------------------------------------------------------------------------------------------
Workflow Name : map-reduce-wf
App Path      : maprfs:/user/mapr/examples/apps/map-reduce/workflow.xml
Status        : SUCCEEDED
Run           : 0
User          : mapr
Group         : -
Created       : 2016-09-09 06:27 GMT
Started       : 2016-09-09 06:27 GMT
Last Modified : 2016-09-09 06:28 GMT
Ended         : 2016-09-09 06:28 GMT
CoordAction ID: -

Actions
------------------------------------------------------------------------------------------------------------------------------------
ID                                                                            Status    Ext ID                 Ext Status Err Code  
------------------------------------------------------------------------------------------------------------------------------------
0000000-160828220950359-oozie-mapr-W@:start:                                  OK        -                      OK         -         
------------------------------------------------------------------------------------------------------------------------------------
0000000-160828220950359-oozie-mapr-W@mr-node                                  OK        job_1473389464865_0002 SUCCEEDED  -         
------------------------------------------------------------------------------------------------------------------------------------
0000000-160828220950359-oozie-mapr-W@end                                      OK        -                      OK         -         
------------------------------------------------------------------------------------------------------------------------------------
```
##mapr-spark
For the Spark installation, I decided to try the Mapr installer to add additional nodes. This worked flawlessly, so I decided to use it instead.
The following Mapr Demos and snippets are ran from the new node, mapredge1.lan

###Mapr-spark-streaming

####Git clone 
```
git clone https://github.com/caroljmcdonald/mapr-streams-spark ~/mapr-streams-spark
```
####Add to Maprfs
```
hadoop fs -put ~/mapr-streams-spark /apps
```
####Create Stream and Topics
```
maprcli stream create -path /apps/mapr-streams-spark/stream -produceperm p -consumeperm p -topicperm p
maprcli stream topic create -path /apps/mapr-streams-spark/stream -topic cdrs -partitions 3
maprcli stream topic create -path /apps/mapr-streams-spark/stream -topic cdrp -partitions 3
```

```
hadoop fs -mkdir /apps/stest
hadoop fs -chmod 0777 /apps/stest
maprcli stream create -path /apps/stest/stream -produceperm p -consumeperm p -topicperm p
maprcli stream topic create -path /apps/stest/stream -topic cdrs -partitions 3
maprcli stream topic create -path /apps/stest/stream -topic cdrp -partitions 3
```

```
hadoop fs -mkdir /apps/st1
hadoop fs -chmod 0777 /apps/st1
maprcli stream create -path /apps/st1/stream -produceperm p -consumeperm p -topicperm p
maprcli stream topic create -path /apps/st1/stream -topic cdrs -partitions 1
maprcli stream topic create -path /apps/st1/stream -topic cdrp -partitions 1
```

####Produce msgs according to git repo:
```
java -cp mapr-streams-spark-1.0.jar:`mapr classpath` com.streamskafka.example.MsgProducer /apps/mapr-streams-spark/stream:cdrs /home/mapr/mapr-streams-spark/sms-call-internet-tn-2013-11-01.txt
```

```
java -cp mapr-streams-spark-1.0.jar:`mapr classpath` com.streamskafka.example.MsgProducer /apps/stest/stream:cdrs /home/mapr/mapr-streams-spark/sms-call-internet-tn-2013-11-01.txt
```

```
java -cp mapr-streams-spark-1.0.jar:`mapr classpath` com.streamskafka.example.MsgProducer /apps/st1/stream:cdrs /home/mapr/mapr-streams-spark/sms-call-internet-tn-2013-11-01.txt
```
###Alternate slower producer using Kafka API:
```
while :; do echo -e "$(($RANDOM % 5))\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM" | /opt/mapr/kafka/kafka-0.9.0/bin/kafka-console-producer.sh --broker-list 1:1 --topic /apps/mapr-streams-spark/stream:cdrs; sleep $(($RANDOM % 5)); done
```

```
while :; do echo -e "$(($RANDOM % 5))\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM" | /opt/mapr/kafka/kafka-0.9.0/bin/kafka-console-producer.sh --broker-list 1:1 --topic /apps/stest/stream:cdrs; sleep $(($RANDOM % 5)); done
```

```
while :; do echo -e "$(($RANDOM % 5))\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM\t$RANDOM" | /opt/mapr/kafka/kafka-0.9.0/bin/kafka-console-producer.sh --broker-list 1:1 --topic /apps/st1/stream:cdrs; sleep $(($RANDOM % 5)); done
```

####Submit the Spark Kafka Consumer Job:
```
/opt/mapr/spark/spark-1.6.1/bin/spark-submit --master yarn-client --class com.sparkkafka.example.SparkKafkaConsumer  mapr-streams-spark-1.0.jar 1:1 /apps/mapr-streams-spark/stream:cdrs /apps/mapr-streams-spark/stream:cdrp
```

```
/opt/mapr/spark/spark-1.6.1/bin/spark-submit --master yarn-client --class com.sparkkafka.example.SparkKafkaConsumer  mapr-streams-spark-1.0.jar 1:1 /apps/stest/stream:cdrs /apps/stest/stream:cdrp
```

```
/opt/mapr/spark/spark-1.6.1/bin/spark-submit --master yarn-client --class com.sparkkafka.example.SparkKafkaConsumer  mapr-streams-spark-1.0.jar 1:1 /apps/st1/stream:cdrs /apps/st1/stream:cdrp
```

####Consume New topic
```
java -cp mapr-streams-spark-1.0.jar:`mapr classpath` com.streamskafka.example.MsgConsumer /apps/mapr-streams-spark/stream:cdrp
```

```
java -cp mapr-streams-spark-1.0.jar:`mapr classpath` com.streamskafka.example.MsgConsumer /apps/stest/stream:cdrp
```

```
java -cp mapr-streams-spark-1.0.jar:`mapr classpath` com.streamskafka.example.MsgConsumer /apps/st1/stream:cdrp
```

##Spark Streaming
###Using the project park-streaming project:

[spark-streaming-mapr-streams Github project](https://github.com/vicenteg/spark-streaming-mapr-streams)
###Run Kafka Producer
```
while :; do fortune | /opt/mapr/kafka/kafka-0.9.0/bin/kafka-console-producer.sh --broker-list 1:1 --topic /tmp/spark-test-stream:topic1; sleep $(($RANDOM % 5)); done
```

###Submit to Spark
```
mapr@mapredge1:~/spark-streaming-mapr-streams$ /opt/mapr/spark/spark-1.6.1/bin/spark-submit --master yarn-client --class com.mapr.example.SparkConsumer target/SparkConsumer-1.0-SNAPSHOT.jar 1:1 /tmp/spark-test-stream:topic1
```

###Using other project's MsgConsumer
```
java -cp mapr-streams-spark-1.0.jar:`mapr classpath` com.streamskafka.example.MsgConsumer /tmp/spark-test-stream:topic1
```




