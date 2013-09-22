---
layout: post
title: "Install Maven on VM with Puppet"
date: 2013-09-22 23:26
comments: true
categories: maven puppet vm vagrant scripts virtualization automation
---
In my [previous] post I've explained how to setup Java on virtual machine using [Puppet]. Now it's a time to make it for maven, basically most of the steps are the same.

<!-- more -->

### 1. Download maven archive.
Visit Apache [Maven site][Maven] and download recent binary tar gz archived maven version.
### 2. Create maven puppet module folders.
Execute following terminal command in your working directory:
{% codeblock Terminal Window %}
mkdir -p modules/maven/{files,manifests}
{% endcodeblock %}
And copy downloaded maven binaries archive (apache-maven-3.1.0-bin.tar.gz in my case) into files sub-folder:
{% codeblock %}
cp apache-maven-3.1.0-bin.tar.gz modules/maven/files/
{% endcodeblock %}
### 3. Write down configuration script.
Create _init.pp_ files inside manifests folder
{% codeblock Terminal Window %}
touch modules/maven/manifests/init.pp
{% endcodeblock %}
Now let's put there following implementation:
{% codeblock lang:ruby%}
class maven {
	$maven_home = "/usr/lib/maven"
	$maven_archive = "apache-maven-3.1.0-bin.tar.gz"
	$maven_folder = "apache-maven-3.1.0"
	
	Exec {
		path => [ "/usr/bin", "/bin", "/usr/sbin"]
	}
	
	file { "/tmp/${maven_archive}":
		ensure => present,
		source => "puppet:///modules/maven/${maven_archive}",
		owner => vagrant,
		mode => 755,
	}
	
	exec { "extract maven":
		command => "tar xfv ${maven_archive}",
		cwd => "/tmp",
		creates => "${maven_home}",
		require => File["/tmp/${maven_archive}"]
	}

	exec { "move maven":
		command => "mv ${maven_folder} ${maven_home}",
		creates => "${maven_home}",
		cwd => "/tmp",
		require => Exec["extract maven"]
	}
	
	file { "/etc/profile.d/maven.sh":
		content =>   "export MAVEN_HOME=${maven_home}
					  export M2=\$MAVEN_HOME/bin
					  export PATH=\$PATH:\$M2
					  export MAVEN_OPTS=\"-Xmx512m -Xms512m\""
	}
}
{% endcodeblock %}
### 4. Put everything together.
I'll continue to use Vagrantfile and manifest from [previous] post, therefore in order to test my new puppet script for maven install I'll just use already create test_vm.pp and add following lines:
{% codeblock Terminal WIndow %}
echo "include maven" >> manifests/test_vm.pp
{% endcodeblock %}
### 5. Reload VM (or start).
{% codeblock Terminal Window %}
vagrant reload test_vm
{% endcodeblock %}
### 6. Enjoy!
 
 In my next post I'll finilize and put together current result with script from my [previous] post in order to implement scripts which will enable to setup Hadoop cluster on VM. Therefore all I've described in my [first] post could be easily automated.

[Puppet]: http://puppetlabs.com/
[Maven]:http://maven.apache.org/download.cgi
[previous]:blog/2013/09/21/install-java-on-vm-using-vagrant-and-puppet
[first]: blog/2013/09/21/setup-hadoop-cluster/