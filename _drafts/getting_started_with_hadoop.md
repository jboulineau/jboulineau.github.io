---
layout: post
title: "Getting Started with Hadoop"
---

### Installation

Following along with the instructions on the [official Hadoop site](https://hadoop.apache.org) is pretty good, but you may run into some of the same questions and issues when I first started. The goal of this post is to help you avoid some of the same frustrations I had.

IF run without configuration, the HDFS filesystem will be added to /tmp/hadoop-{username}/dfs/name

ERROR: Attempting to operate on hdfs namenode as root
ERROR: but there is no HDFS_NAMENODE_USER defined. Aborting operation.
Starting datanodes
ERROR: Attempting to operate on hdfs datanode as root
ERROR: but there is no HDFS_DATANODE_USER defined. Aborting operation.
ERROR: JAVA_HOME is not set and could not be found.

# Resolution: add to hadoop-env.sh

sudo mkdir /home/hdfs

Problem: pdsh@surface-ubuntu: localhost: connect: Connection refused
Solution: export PDSH_RCMD_TYPE=ssh to hadoop-env.sh

Problem: localhost: root@localhost: Permission denied (publickey,password).
Solution: create ssh key for root, add to authorized_keys

localhost: root@localhost: Permission denied (publickey,password).

/opt/hadoop-3.1.0/bin/hdfs: 26: /opt/hadoop-3.1.0/bin/hdfs: function: not found
/opt/hadoop-3.1.0/bin/hdfs: 28: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_option: not found
/opt/hadoop-3.1.0/bin/hdfs: 29: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_option: not found
/opt/hadoop-3.1.0/bin/hdfs: 30: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_option: not found
/opt/hadoop-3.1.0/bin/hdfs: 31: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_option: not found
/opt/hadoop-3.1.0/bin/hdfs: 32: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_option: not found
/opt/hadoop-3.1.0/bin/hdfs: 33: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_option: not found
/opt/hadoop-3.1.0/bin/hdfs: 35: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 36: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 37: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 38: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 39: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 40: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 41: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 42: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 43: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 44: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 45: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 46: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 47: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 48: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 49: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 50: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 51: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 52: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 53: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 54: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 55: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 56: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 57: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 58: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 59: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 60: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 61: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 62: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 63: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 64: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 65: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 66: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 67: /opt/hadoop-3.1.0/bin/hdfs: hadoop_add_subcommand: not found
/opt/hadoop-3.1.0/bin/hdfs: 68: /opt/hadoop-3.1.0/bin/hdfs: hadoop_generate_usage: not found
/opt/hadoop-3.1.0/bin/hdfs: 76: /opt/hadoop-3.1.0/bin/hdfs: function: not found
/opt/hadoop-3.1.0/bin/hdfs: 213: /opt/hadoop-3.1.0/bin/hdfs: hadoop_validate_classname: not found
/opt/hadoop-3.1.0/bin/hdfs: 214: /opt/hadoop-3.1.0/bin/hdfs: hadoop_exit_with_usage: not found
/opt/hadoop-3.1.0/bin/hdfs: 221: /opt/hadoop-3.1.0/bin/hdfs: [[: not found
/opt/hadoop-3.1.0/bin/hdfs: 230: /opt/hadoop-3.1.0/bin/hdfs: [[: not found
ERROR: Cannot execute /opt/hadoop-3.1.0/bin/../libexec/hdfs-config.sh.

Look in the log:

2018-05-22 17:00:50,524 INFO org.apache.hadoop.util.ExitUtil: Exiting with status 1: org.apache.hadoop.hdfs.server.common.InconsistentFSStateException: Directory /tmp/hadoop-root/dfs/name is in an inconsistent state: storage directory does not exist or is not accessible.



<!-- TODO

Write a script to automate

-->