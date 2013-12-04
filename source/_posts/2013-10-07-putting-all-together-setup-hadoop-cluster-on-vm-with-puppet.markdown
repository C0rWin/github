`---
layout: post
title: "Putting all together setup, Hadoop cluster on VM with Puppet."
date: 2013-09-30 00:14
comments: true
categories: hadoop cluster puppet scripts automation virtualization
---

In my two previous blog post I've explained:

1. [Setup Hadoop cluster][first] - manual configuration on virtual machines which needed to setup Hadoop cluster.
2. [Install Java on VM using vagrant and puppet][second] - I've started to explain how to leverage vagrant and puppet in order to have Java installed in virtual machine.

Now in this post I'll try to explain how-to setup hadoop cluster on virtual environment using vagrant and puppet scripts, so all actions in my first post could be easily automated.

 <!-- more -->
 
 I'm going to use my first two posts as a reference and assume you have downloaded all files needed, moreover you already have puppet module script for java install and configuration.
 
Start as usual:

### 1. Create module folders:
{% codeblock Terminal Widow %}
mkdir -p modules/hadoop/{files,manifests,templates}
{% endcodeblock %}

###2. Copy hadoop files into module folder:
As I said I'm going to use files from previous [post][first]
{% codeblock Terminal Window %}
cp hadoop-1.2.1.tar.gz modules/hadoop/files/
{% endcodeblock %}

###3. Write implementation of hadoop module for puppet.
Following script will download, unpack and setup hadoop on desired amount of virtual machines.

{% codeblock init.pp lang:ruby %}
class hadoop( $masterNode, $slaveNodes, $distrFile, $hadoopHome) {

    Exec {
        path => [ "/usr/bin", "/bin", "/usr/sbin"]
    }

    file { "/tmp/${distrFile}.tar.gz" :
        source => "puppet:///modules/hadoop/${distrFile}.tar.gz",
        owner =>  vagrant,
        mode  => 755,
        ensure => present
    }
    
    exec { "extract distr":
        cwd => "/tmp",
        command => "tar xf ${distrFile}.tar.gz",
        creates => "${hadoopHome}",
        user => vagrant,
        require => File["/tmp/${distrFile}.tar.gz"]
    }
    
    exec { "move distr":
        cwd => "/tmp",
        command => "mv ${distrFile} ${hadoopHome}",
        creates => "${hadoopHome}",
        user => vagrant,
        require => Exec["extract distr"]
    }

    file { "/etc/profile.d/hadoop.sh":
        content => "export HADOOP_PREFIX=\"${hadoopHome}\"
        export PATH=\"\$PATH:\$HADOOP_PREFIX/bin\""
    }

    file { "${hadoopHome}/conf/slaves":
        content => template( "hadoop/slaves.erb"),
        mode => 644,
        owner => vagrant,
        group => vagrant,
        require => Exec[ "move distr"]
    }
    
    file { "${hadoopHome}/conf/masters":
        content => template( "hadoop/masters.erb"),
        mode => 644,
        owner => vagrant,
        group => vagrant,
        require => Exec[ "move distr"]
    }

    file { "${hadoopHome}/conf/core-site.xml":
        content => template( "hadoop/core-site.erb"),
        mode => 644,
        owner => vagrant,
        group => vagrant,
        require => Exec[ "move distr"]
    }

    file { "${hadoopHome}/conf/hdfs-site.xml":
        content => template( "hadoop/hdfs-site.erb"),
        mode => 644,
        owner => vagrant,
        group => vagrant,
        require => Exec[ "move distr"]
    }

    file { "${hadoopHome}/conf/mapred-site.xml":
        content => template( "hadoop/mapred-site.erb"),
        mode => 644,
        owner => vagrant,
        group => vagrant,
        require => Exec[ "move distr"]
    }
}
{% endcodeblock %}

###4. Write puppet templates for configuration files:
You probably paid attention that in order to setup hadoop configuration files I've used puppet templates function (you can read more [here][template]). Therefore now we need to provide three configuration files in order to complete that part.

{% codeblock masters lang:ruby %}
<% @masterNode %>
{% endcodeblock %}

{% codeblock slaves lang:ruby %}
<% @slaveNodes.each do |node| -%>
<% node %>
<% end -%>
{% endcodeblock %}

{% codeblock core-site.erb lang:xml %}
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
        <property>
            <name>fs.default.name</name>
            <value><%= @masterNode %>:9000</value>
            <description>The name of the default file system. A URI whose scheme and authority determine the FileSystem implementation.</description>
        </property>
    </configuration>
{% endcodeblock %}

{% codeblock  hdfs-site.erb lang:xml %}
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
        <description>The actual number of replications can be specified when the file is created.</description>
    </property>
</configuration>
{% endcodeblock %}

{% codeblock mapred-site.erb lang:ruby %}
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value><% @masterNode %>:9001</value>
        <description>The host and port that the MapReduce job tracker runs at.</description>
    </property>
</configuration>
{% endcodeblock %}


Where:

* __$masterNode__ - IP address of hadoop master node.
* __$slaveNodes__  -  list of IP addresses for hadoop slave nodes.
* __$distrFile__ - hadoop distribution file name.
* __$hadoopHome__ - desired hadoop home folder.

###5. Next step is to write Vagrant file.
{% codeblock lang:ruby Vagrant %}
Vagrant::Config.run do |config|
  config.vm.define :hadoop_master do |hadoop_config|
        hadoop_config.vm.box = "ubuntu"
        hadoop_config.vm.network :hostonly, "192.168.32.1"
        hadoop_config.vm.host_name = "master"
        hadoop_config.vm.customize [ "modifyvm", :id, "--memory", "1024"]
        hadoop_config.vm.provision :puppet, :facter => { "fqdn" => "master"} do |puppet|
                puppet.module_path = "modules"
                puppet.manifests_path = "manifests"
                puppet.manifest_file  = "hadoop.pp"
        end
  end

  config.vm.define :hadoop_slave do |hadoop_config|
        hadoop_config.vm.box = "ubuntu"
        hadoop_config.vm.network :hostonly, "192.168.32.2"
        hadoop_config.vm.host_name = "slave"
        hadoop_config.vm.customize [ "modifyvm", :id, "--memory", "1024"]
        hadoop_config.vm.provision :puppet, :facter => { "fqdn" => "slave"} do |puppet|
                puppet.module_path = "modules"
                puppet.manifests_path = "manifests"
                puppet.manifest_file  = "hadoop.pp"
        end
  end
end
{% endcodeblock %}

###6. Now we need to write some code to combine java installation process with hadoop setup.
Following script will initialize java defaults and hadoop installation.
{% codeblock lang:ruby hadoop.pp %}
class { "hadoop":
    masterNode => "192.168.32.1",
    slaveNodes => ["192.168.32.1", "192.168.32.2"],
    distrFile  => "hadoop-1.2.1",
    hadoopHome => "/home/vagrant/hadoop"
}
include java
{% endcodeblock %}

###7. Run virtual machines.

{% codeblock Terminal Window %}
vagrant up hadoop_{master,slave}
{% endcodeblock %}

Now if we proceed to the last step in [hadoop][first] post and will try to run command on master node:
{% codeblock Terminal Window %}
sudo ./hadoop namenode -format
sudo ./start-all.sh
{% endcodeblock %}
We will notice that we have forgot to configure and publish ssh keys between nodes, hence cluster is not able to properly startup. Therefore we need to continue a bit more.

###8. Setup ssh keys.
Run in terminal:
{% codeblock Terminal Window %}
ssh-keygen -t rsa
{% endcodeblock %}

and provide key path up to your files module folder ("modules/hadoop/files/id_rsa" for example). Enter your passphrase, now open new generated id_rsa.pub and copy into clipboard newly generated public key.
Now we need to add following lines into our module init.pp file:

{% codeblock lang:ruby init.pp %}
    file { "/home/vagrant/.ssh/id_rsa":
        source => "puppet:///modules/hadoop/id_rsa",
        mode => 600,
        owner => vagrant,
        group => vagrant,
    }

    file { "/home/vagrant/.ssh/id_rsa.pub":
        source => "puppet:///modules/hadoop/id_rsa.pub",
        mode => 600,
        owner => vagrant,
        group => vagrant,
    }

    ssh_authorized_key { "ssh_key":
        ensure => "present",
        key => "past here content from your clipboard (public key value from id_rsa.pub file)",
        type => "ssh-rsa",
        user => vagrant,
        require => File["/home/vagrant/.ssh/id_rsa.pub"]
    }
{% endcodeblock %}

###9. Enjoy.

Now you can reload virtual machines

{% codeblock Terminal Window %}
vagrant reload hadoop_{master,slave}
{% endcodeblock %}

then loggin into master node

{% codeblock Terminal Window %}
vagrant ssh hadoop_master
{% endcodeblock %}

and finally start cluster

{% codeblock Terminal Window %}
sudo ./hadoop namenode -format
sudo ./start-all.sh
{% endcodeblock %}

You can find sources on my [github].

[first]: blog/2013/09/21/setup-hadoop-cluster/
[second]: blog/2013/09/21/install-java-on-vm-using-vagrant-and-puppet
[template]:http://docs.puppetlabs.com/guides/templating.html
[github]:https://github.com/C0rWin/hadoop-puppet