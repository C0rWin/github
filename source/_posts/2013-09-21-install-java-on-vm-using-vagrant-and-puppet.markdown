---
layout: post
title: "Install Java on VM using Vagrant and Puppet"
date: 2013-09-21 23:25
comments: true
categories: java vagrant vm puppet scripts virtualization automation
---
As part of my work I need to be able to test different Java Web applications. I need to be able to run some tests on more then only single computer, hence I need rapid way of setting up several Linux based computers/servers with configured Java running, usually I'm using Ubuntu based destributives.

Therefore I've choose the winning couple of [Puppet] and [Vagrant]. You're more than welcome to read about these tools and I'll skip the installation guide for them, since google has pleanty of different tutorials explaning how to do it for different platform, so you just need to pick the right one for you.

<!-- more -->

I'm going to show how to leverage [Puppet] modules architecture, how to implement new module which will be capable to install and configure Java for new VM machine. To learn more about modules in [Puppet], please take a look on [modules fundamentals][puppet_modules] article on [Puppet] labs site.

### 1. Download Java.
Very first step which has to be done, is to download tar gz archive of Java JDK or (JRE), from Oracle official [site][Oracle]. By the time I'm writting this post current lates JDK tar gz is _jdk-7u40-linux-i586.tar.gz_.
### 2. Create new Puppet module.
{% codeblock Terminal Window %}
mkdir -p modules/java/{files,manifests}
{% endcodeblock %}
### 3. Copy Java JDK into module folders.
{% codeblock Terminal Window %}
cp jdk-7u40-linux-i586.tar.gz modules/java/files/
{% endcodeblock %}
### 4. Write module implementation.
+ Create _init.pp_ file within __manifests__ folder
{% codeblock Terminal WIndow %}
touch modules/java/manifests/init.pp
{% endcodeblock %}
+ Now we need to write down following content:
{% codeblock init.pp lang:ruby  %}
class java {
    $java_archive = "jdk-7u40-linux-x64.tar.gz"
    $java_home = "/usr/lib/jvm/jdk1.7.0_40/"
    $java_folder = "jdk1.7.0_40"

    Exec {
        path => [ "/usr/bin", "/bin", "/usr/sbin"]
    }

    file { "/tmp/${java_archive}" :
        ensure => "present",
        source => "puppet:///modules/java/${java_archive}",
        owner  => vagrant,
        mode   => 755
    }

    exec { 'extract jdk':
        cwd => '/tmp',
        command => "tar xfv ${java_archive}",
        creates => $java_home,
        require => File["/tmp/${java_archive}"],
    }

    file { '/usr/lib/jvm' :
        ensure => directory,
        owner => vagrant,
        require => Exec['extract jdk']
    }

    exec { 'move jdk':
        cwd => '/tmp',
        creates => $java_home,
        require => File['/usr/lib/jvm'],
        command => "mv ${java_folder} /usr/lib/jvm/"
    }

    exec {'install java':
        require => Exec['move jdk'],
        logoutput => true,
        command => "update-alternatives --install /bin/java java ${java_home}/bin/java 1"
    }

    exec {'set java':
        require => Exec['install java'],
        logoutput => true,
        command => "update-alternatives --set java ${java_home}/bin/java"
    }

    exec {'install javac':
        require => Exec['move jdk'],
        logoutput => true,
        command => "update-alternatives --install /bin/javac javac ${java_home}/bin/javac 1"
    }

    exec {'set javac':
        require => Exec['install javac'],
        logoutput => true,
        command => "update-alternatives --set javac ${java_home}/bin/javac"
    }
    
	file { "/etc/profile.d/java.sh":
		content => "export JAVA_HOME=${java_home}
				  	 export PATH=\$PATH:\$JAVA_HOME/bin"
	}
}
{% endcodeblock %}
### 5. Run and test new module.

+ Run:
{% codeblock Terminal Window %}
vagrant init
{% endcodeblock %}
+ Edit Vagrantfile, put following content:
{% codeblock Vagrantfile lang:ruby %}
Vagrant::Config.run do |config|
  config.vm.define :test_vm do |test_vm|
        test_vm.vm.box = "ubuntu"
        test_vm.vm.network :hostonly, "10.71.71.10"
        test_vm.vm.customize [ "modifyvm", :id, "--memory", "1024"]
        test_vm.vm.provision :puppet do |puppet|
                puppet.module_path = "~/vagrant/manifests/"
                puppet.module_path = "modules"
                puppet.manifests_path = "manifests"
                puppet.manifest_file  = "test_vm.pp"
        end
  end
end
{% endcodeblock %}
+ Create _manifests_ folder and _test_vm.pp_:
{% codeblock Terminal Window %}
mkdir manifests && touch manifests/test_vm.pp
echo "include java" >> manifests/test_vm.pp
{% endcodeblock %}
+ Run VM 
{% codeblock Terminal Window %}
vagrant up test_vm
{% endcodeblock %}

### 6. Enjoy!

[Puppet]:http://puppetlabs.com/
[puppet_modules]: http://docs.puppetlabs.com/puppet/2.7/reference/modules_fundamentals.html
[Vagrant]:http://www.vagrantup.com/
[Oracle]:http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html
