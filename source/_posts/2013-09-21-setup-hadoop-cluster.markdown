---
layout: post
title: "Setup hadoop cluster"
date: 2013-09-21 01:10
comments: true
categories: hadoop how-to cluster distributed
---
Recently I've required to execute some heavy clustering computation on relatively big dataset. Since [Mahout][mahout] (scalable machine learning framework) already has all required capabilities and holds implementation of base clustering algorithm, I've decide to use it as a start point and bacause [Mahout][mahout] is [Hadoop][hadoop] based I've had to setup cluster of [Hadoop][hadoop] nodes to be able to execute my clustering task.

So here I'll try to memorize steps which required for distributed setup of hadoop cluster, for sake of simplicity I'll described setup for only two nodes: master and slave. In this blog post I am going to describe manual install and configuration, while in the next I'll describe the automation configuration and install using [puppet][puppet] and [vagrant][vagrant] tools.  I will describe the installation process in context of Ubuntu 12.10 server, while I belive same steps will work for other distirbutives as well.
<!-- more -->
Here is the steps required for Hadoop install and configuration in order to be able to execute distributed tasks on cluster nodes:

1. [Java download and installation][java_install].
2. [Download Hadoop and setup][hadoop_install].
3. [Setup ssh configuration and configure public keys][ssh].
4. [Hadoop startup][hadoop_startup].

### <a id="java_download_install">Java download and installation.</a>
Bellow steps has to be performed for each node in cluster in order to make sure each one has recent Oracle Java JDK installed.

+ Download recent Java JDK for Linux distribution from Oracle official [site][oracle]. By the time writing this post recent tar gz was *jdk-7u40-linux-i586.tar.gz*.
+ Extract files from archive, run following command in terminal:
{% codeblock %}
tar xfv jdk-7u40-linux-i586.tar.gz
{% endcodeblock %}
+ Move extracted folder:
{% codeblock %}
sudo mv jdk1.7.0_40 /usr/lib/jvm/jdk1.7.0_40
{% endcodeblock %}
+ Update alternatives, install setup aliases for new Java jdk:
{% codeblock %}
sudo update-alternatives --install /bin/java java /usr/lib/jvm/jdk1.7.0_40/bin/java 1
sudo update-alternatives --install /bin/javac javac /usr/lib/jvm/jdk1.7.0_40/bin/javac 1
{% endcodeblock %}
+ Make installed aliases active:
{% codeblock %}
sudo update-alternatives --set java /usr/lib/jvm/jdk1.7.0_40/bin/java
sudo update-alternatives --set javac /usr/lib/jvm/jdk1.7.0_40/bin/javac
{% endcodeblock %}
+ Configure __$JAVA_HOME__:
{% codeblock %}
sudo touch /etc/profile.d/java.sh
sudo -c 'echo "export JAVA_HOME=/usr/lib/jvm/jdk1.7.0_40/" >> /etc/profile.d/java.sh'
sudo -c 'echo "export PATH=$PATH:$JAVA_HOME/bin" >> /etc/profile.d/java.sh' 
{% endcodeblock %}

### <a id="hadoop_install">Download Hadoop distribution</a>

+ Download Hadoop from releases [site][hadoop_site], choose last stable version.
+ Extract downloaded files into home folder:
{% codeblock %} 
tar xfv hadoop-1.2.1.tar.gz
{% endcodeblock %}
+ Configure __$HADOOP_HOME__:
{% codeblock %}
sudo touch /etc/profile.d/hadoop.sh
sudo -c 'echo "export HADOOP_HOME=~/hadoop-1.2.0/" >> /etc/profile.d/hadoop.sh'
sudo -c 'echo "export PATH=$PATH:$HADOOP_HOME/bin" >> /etc/profile.d/hadoop.sh' 
{% endcodeblock %}

Now for next few steps assume we have two computer with IP addresses as follow: **192.168.17.1** (master) and **192.168.17.2** (slave).

+ Open file $HADOOP_HOME/conf/core-site.xml and write content:
{% codeblock core-site.xml lang:xml%}
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
        <property>
            <name>fs.default.name</name>
            <value>hdfs://192.168.17.1:9000</value>
            <description>The name of the default file system. A URI whose scheme and authority determine the FileSystem implementation.</description>
        </property>
    </configuration>
{% endcodeblock %}
+ Next open $HADOOP_HOME/conf/hdfs-site.xml and write content:
{% codeblock hdfs-site.xml lang:xml%}
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
        <description>The actual number of replications can be specified when the file is created.</description>
    </property>
</configuration>
{% endcodeblock%}
+ Now open $HADOOP_HOME/conf/mapred-site.xml and write content:
{% codeblock mapred-site.xml lang:xml %}
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value>192.168.17.1:9001</value>
        <description>The host and port that the MapReduce job tracker runs at.</description>
    </property>
</configuration>
{% endcodeblock %}
+ And finally you need to change two more files first $HADOOP_HOME/conf/masters and second is $HADOOP_HOME/conf/slaves, not too hard to guess what should be contet of each one of these:
{% codeblock masters %}
192.168.17.1
{% endcodeblock %}

and

{% codeblock slaves %}
192.168.17.1
192.168.17.2
{% endcodeblock %}

I've putted 192.168.17.1 in both files, since I'd like to have master node to execute computational task as well and hold distributed data.

Now we proceed to the final steps of ssh configuration and actuall Hadoop startup.

### <a id="ssh">Setup ssh configuration and configure public keys</a>

Configure master:

{% codeblock Generate public key for master node %}
ssh-key -t rsa
{% endcodeblock %}

{% codeblock Copy public key to slave node%}
cat ~/.ssh/id_rsa.pub | ssh 192.168.17.2 'cat >> .ssh/authorized_keys'
{% endcodeblock %}

Now same for slave:

{% codeblock Generate public key for slave node %}
ssh-key -t rsa
{% endcodeblock %}

{% codeblock Copy public key to master node%}
cat ~/.ssh/id_rsa.pub | ssh 192.168.17.1 'cat >> .ssh/authorized_keys'
{% endcodeblock %}

### <a id="hadoop_startup">Hadoop startup</a>

Now after we have complete all configurations we need to run following commands in terminal on master (192.168.17.1) node:

{% codeblock %}
sudo ./hadoop namenode -format
sudo ./start-all.sh
{% endcodeblock %}


[Here](http://hadoop.apache.org/docs/stable/cluster_setup.html) you can find more details and explanations on how to configure and setup Hadoop cluster.

Obviously it's ridiculous to proceed all these steps each time I need to setup new Hadoop cluster, so in my next blog post I'll write how-to setup Hadoop cluster using [vagrant] and [puppet] to enable automation of this procedure.

[java_install]: #java_download_install
[ssh]: #ssh

[hadoop_install]: #hadoop_install
[hadoop_config]: #hadoop_config
[hadoop_startup]: #hadoop_startup
[hadoop_site]: http://apache.spd.co.il/hadoop/common/
[hadoop]: http://hadoop.apache.org/

[oracle]: http://www.oracle.com/technetwork/java/javase/downloads/index.html

[mahout]: http://mahout.apache.org/
[puppet]: http://puppetlabs.com/
[vagrant]: http://www.vagrantup.com/
