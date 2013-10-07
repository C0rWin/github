---
layout: post
title: "Running Matlab code on Amazon cloud."
date: 2013-10-08 00:04
comments: true
categories: matab amazon distributed ec2
---
Recently I've bumped into the need to have Matlab code running on Amazon cloud infrastructure. Moreover my Matlab application has to communicate with rest of the world, therefore I have  had to find some third party communication labrary for Matlab which will do the work. It's long story why do I need this, but googling around I've found series of the blog entries, which explains in details how to do it, basically it also explains the reason why do I need to do it.

1. [Using Amazon EC2 to speed up matlab optimisation][first]
2. [Writing a socket interface in Matlab to send / receive the commands][second]
3. [Setting up an Ubuntu EC2 instance to run the compiled Matlab code][third]
4. [Writing matlab software to communicate with the server.][fourth]

As you may probably guess were this strightforward procees I've never been spending my time to write this blog entry. Since I've had to solve a couple of puzzles and tackled with quite a few trick I'd like to memorize my expirience here, hopefully it will save time to more people later.

<!-- more -->

###1. First step, download msocket toolbox(set of wrappers for socket communication) .
Msocket library located at [google code repository][msocket] and looks like nobody has touched for year or two and there were several changes in Matlab API since then. Therefore you will have to patch a bit code in this library to make it compile. 
#####1.1 Add include to matrix module (#include <matrix.h>) for msrecv.cpp and msrecvraw.c.
#####1.2 Replace each occurence of __"mxCreateScalarDouble"__ with __"mxCreateDoubleScalar"__.
#####1.3 Open msocket with Matlab and execute compileit, this will compile source code for you.
#####1.4 Add msocket library to Matlab path.
Now you ready to implement your client/server application while leveraging mathematics capabilities of Matlab.

###2. Implement your client/server application.
I will skip this part, mainly because it's greatly covered in blog post series I've mentioed above and because due to licensing regulation I cannot past this code here in the blog.

###3. Instantiate EC2 on Amazon.
Just pick instance which suites your goals and run them. I've used micro instance with Ubuntu 12.04 LTE as OS.

###4. Update packages.
SSH into your newly created instances and run form terminal an update:

{% codeblock Terminal Window %}
sudo apt-get update && sudo apt-get upgrade -y
{% endcodeblock %}

Once update install several dependencies we will use later.

{% codeblock Terminal Window %}
sudo apt-get install g++ libxmu6 libxt6 -y
{% endcodeblock %}

###5. Download MCR.
Download MCR from Matlab site [here][mcr]. Once download is finished upload it on your EC2 instances.

###6.Install MCR at EC2.
On each instance run in terminal:

{% codeblock Terminal Window %}
cd MCR_R2013a_glnxa64_installer/
./install -console -mode silent  -agreeToLicense yes
{% endcodeblock %}

###7. Compile your Matlab application.

{% codeblock Matlab lang:fortran %}
mcc -m NameOfYourApplication
{% endcodeblock %}

###8. Upload compiled on EC2 instances.
After compilation has finished you need to upload compiled code onto the cloud instances. You need to upload files:
* NameOfYourApplication
* run_NameOfYourApplication.sh

###9. Run your app.
{% codeblock Terminal Window %}
./run_NameOfYourApplication.sh /usr/local/MATLAB/MATLAB_Compiler_Runtime/v81/
{% endcodeblock %}

###10.Enjoy!

[first]:http://noisyaccumulation.blogspot.co.il/2010/09/using-amazon-ec2-to-speed-up-matlab.html
[second]:http://noisyaccumulation.blogspot.co.il/2010/12/using-amazon-ec2-to-speed-up-matlab.html
[third]:http://noisyaccumulation.blogspot.com/2010/12/using-amazon-ec2-to-speed-up-matlab_22.html
[fourth]:http://noisyaccumulation.blogspot.com/2010/12/using-amazon-ec2-to-speed-up-matlab_9326.html
[mcr]:http://www.mathworks.com/products/compiler/mcr/
[msocket]:http://code.google.com/p/msocket/