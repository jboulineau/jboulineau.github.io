---
layout: post
title: "Running Hadoop on WSL"
date: 2018-05-23
categories: "Hadoop"
excerpt: "I was curious if it was possible to get Hadoop running under Windows Subsystem for Linux. It is."
---

I've been using Linux a lot lately, but it's become annoying switching back-and-forth between Windows and the Linux VM. I've been thinking about switching to Linux as my primary OS, which I may still do, but I thought I'd give [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/about) a go to see how far I could get with it. Trying to get Hadoop to run was as good a test as any. It turns out it wasn't all that difficult and I'm far from a Linux expert.

If you want to reproduce my results, follow along with the official Single Cluster [getting started doc](https://hadoop.apache.org/docs/r3.1.0/hadoop-project-dist/hadoop-common/SingleCluster.html) on the [Hadoop site](https://hadoop/apache.org). Make sure you use the one for the version you want to install or else you'll stumble over things like the port that the namenode admin site runs on (50070 with Hadoop 2 and 9870 for 3). I chose 3.1.0.

Installing WSL is as easy as going through the Windows store and selecting the distribution of your choice. I chose Ubuntu 18.04. Once WSL is installed, you need to make it usable. The color scheme is a monstrosity of illegibility. Here are some good [instructions](https://medium.com/@iraklis/fixing-dark-blue-colors-on-windows-10-ubuntu-bash-c6b009f8b97c) on how to get to something that won't make your eyes bleed.

You'll be spending some time in Vi, so let's make that useable to. Add the following line to ~/.vimrc, or else you'll be squinting at an illegible dark blue on black color scheme:

```bash
set background=dark
```

Now, let's get to what I discovered. Below I'm taking the approach of filling in the gaps of the 'Getting Started' doc. Reading this post first should (hopefully) get you over the hurdles you'll run into while you're following the doc. 

## Installing dependencies

Hadoop's main dependency is Java, which is not installed by default in WSL. [Here](https://www.digitalocean.com/community/tutorials/how-to-install-java-with-apt-get-on-ubuntu-16-04) are some straight forward instructions on how to install it. Note that your JAVA_HOME will almost certainly be different than the Getting Started docs. The best way I know how to check is to run:

```bash
update-alternatives --config java
```

That will show you something like:

```bash
/usr/lib/jvm/java-8-oracle/jre/bin/java
```

So, for me, JAVA_HOME should be set to `/usr/lib/jvm/java-8-oracle/` in `etc/hadoop-env.sh`.

## SSH issues

Attempting to connect to localhost via ssh was the only problem I ended up running into. When doing so, I got:

```
Connection closed by 127.0.0.1 port 22
```

It turned out, the necessary keys for sshd to function properly are not automagically generated in WSL (I think). I'm sure you could go through and generate them yourself, but I found it easier just to reinstall  openssh-server. 

```bash
sudo apt purge openssh-server
sudo apt install openssh-server
sudo service ssh start
```

And that's it. If you do the above and then follow along with the 'Getting Started' doc, you should be good to go. 

One more lesson learned.  When running `start-dfs.sh` at one point I got the following error:

```bash
Starting namenodes on [localhost]
localhost: jon@localhost: Permission denied (publickey,password).
Starting datanodes
localhost: jon@localhost: Permission denied (publickey,password).
Starting secondary namenodes [JON-PC]
JON-PC: jon@jon-pc: Permission denied (publickey,password).
```

I had deviated from the instructions by adding a password to my ssh key (for added securities!). That was not good for Hadoop. There MUST be a way to start Hadoop with a password protected key, but I haven't researched that yet. It was easy enough to correct by running the following to remove the password.

```bash
ssh-keygen -p
```