---
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
mkdir -p modules/hadoop/{files,manifests}
{% endcodeblock %}

###2. Copy hadoop files into module folder:
As I said I'm going to use files from previous [post][first]
{% codeblock %}
cp hadoop-1.2.1.tar.gz modules/hadoop/files/
{% endcodeblock %}




[first]: blog/2013/09/21/setup-hadoop-cluster/
[second]: blog/2013/09/21/install-java-on-vm-using-vagrant-and-puppet
