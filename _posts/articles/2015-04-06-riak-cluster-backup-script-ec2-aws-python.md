---
layout: post
title: "Riak Cluster Backup Script on Amazon EC2"
excerpt: "Python script to backup riak cluster on EC2"
category: [databases]
tags: [riak,nosql]
comments: true
share: true
---

I recently wrote about [Understanding Riak Clusters](http://www.cynnefo.com/articles/Understanding-Riak-Clusters-Backup-strategy/) and designing a backup strategy. One of our customer has a 5 node Riak cluster running on AWS EC2 and we had to create a backup job for it. If you are running riak `enterprise` edition, the best way to do a backup is to do a [full sync replication](http://docs.basho.com/riakee/latest/cookbooks/Multi-Data-Center-Replication-Architecture/#Fullsync-Replication) every day to a node in a different datacenter. Since we are not running enterprise edition, we decided to go with file system level backups of each node. Since we were running on Amazon EC2, the ebs snapshots feature comes in handy and it is faster compared to `rsync` or archiving etc. 

The script iterates through a list of nodes and does the following:
	
1. Makes an SSH connection to the node using `fabric`
2. Stops Riak service by running `riak stop` command. Since our storage backend is `leveldb` and not `bitcask`, stopping services is necessary before initiating a snapshot.
3. Takes snapshots of all ebs volumes attached to the instance
4. Starts riak service post snapshot using `riak start` command
5. Checks if all the primary vnodes are up and running using `riak-admin transfers` command. If they aren't, you'll generally see a text like this - *does not have \d+ primary partitions running*
6. Checks if there are `handoffs` pending and waits till they are done before moving on to next node. When a node is down in a riak cluster, vnodes from the other live nodes temporarily takes responsibility for some data and once node is back online, returns the data to original owner. This is a called a [hinted handoff](http://docs.basho.com/riak/latest/ops/running/handoff/). We need to make sure that there are handoff transfers before moving on to the other nodes. Makes sure that riak kv service is up. 
7. Moves to other node and starts from step #1 and at the end, sends out an email with status. 
	
Considerations before running this script:
	
1. You need fabric, boto python libraries
2. Fabric executes remote sudo commands for stopping and starting riak. You need to edit the sudoers file and change `requiretty` to `!requiretty`. [It apparently provides no additional security benefit](http://unix.stackexchange.com/a/122624) and can be removed
	

Blow is the script. You may also download it from [here](https://www.dropbox.com/s/vecmh49hmy4dvqt/riakbackup.py?dl=0): 

{% highlight python %}
import fabric.api as fab
from fabric.api import warn_only
from fabric.network import disconnect_all
from contextlib import contextmanager
from fabric import exceptions
from boto.ec2.connection import EC2Connection
from boto.regioninfo import *
import time, smtplib, sys, re

riak_nodes = {'i-xxxxx1':'ec2-xx-xxx-xx-xx.us-west-1.compute.amazonaws.com',
'i-xxxxx2':'ec2-xx-xxx-xx-xx.us-west-1.compute.amazonaws.com',
'i-xxxxx3':'ec2-xx-xxx-xx-xx.us-west-1.compute.amazonaws.com',
'i-xxxxx4':'ec2-xx-xxx-xx-xx.us-west-1.compute.amazonaws.com',
'i-xxxxx5':'ec2-xx-xxx-xx-xx.us-west-1.compute.amazonaws.com'}


@contextmanager
def ssh(settings):
	with settings:
		try:
			yield
		finally:
			disconnect_all()


def ssh1(host, user, ssh_key, command):
    with ssh(fab.settings(host_string=host, user=user, key_filename=ssh_key, warn_only=True)):
        return fab.sudo(command, pty=False)

def ebs_snapshot(instance_id):
	access_key = 'AWSAccessKeyHere'
	secret_key = 'AWSSecurityKeyHere'
	snapshotname = 'Riak Backup - ' + instance_id + '- ' + time.strftime("%Y-%m-%d %H:%M:%S")
	try:
		ec2_region = boto.ec2.get_region(aws_access_key_id=access_key, aws_secret_access_key=secret_key, region_name='us-east-1')
		print 'Connected Successfully!'
	except Exception, e:
		print e
		print 'Connection failed!'
	
	ec2_conn = boto.ec2.connection.EC2Connection(
    aws_access_key_id=access_key, 
    aws_secret_access_key=secret_key,
    region=ec2_region)	
	volumes = ec2_conn.get_all_volumes(volume_ids=None, filters=None)
	instance_volumes = [v for v in volumes if v.attach_data.instance_id == instance_id]
	for vol in instance_volumes:
		snapshot = ec2_conn.create_snapshot(vol.id, snapshotname)


def sendmail(subject, text):
	user = "vigilante@gmail.com"
	password = "YourSuperSecretPassword"
	FROM = "riakadmin@candyland.com"
	TO = ['dba@canduland.com']
	
	message = """\From: %s\nTo: %s\nSubject: %s\n\n%s
	""" % (FROM, ", ".join(TO), subject, text)
	try:
		server = smtplib.SMTP("smtp.gmail.com", 587)
		server.ehlo()
		server.starttls()
		server.login(user, password)
		server.sendmail(FROM, TO, message)
		server.close()
	except Exception, e:
		print e	


def stop_riak(host, user, ssh_key):
	command = "riak stop"
	output = ssh1(host, user, ssh_key, command)
	if not 'ok' in output:
		subject = "Alert! Riak Backup Error"
		text = "Riak service failed to STOP on host %s. Terminating the backup script!" %host
		sendmail(subject, text)
		sys.exit()

def start_riak(host, user, ssh_key):
	command = "riak start"
	ssh1(host, user, ssh_key, command)
	#run command again to verify if node is running
	output = ssh1(host, user, ssh_key, command)
	output.replace('\n', ' ').replace('\r', '')	
	if not 'Node is already running!' in output:
		subject = "Alert! Riak backup Error"
		text = "Riak Service failed to START on host %s. Terminating the backup script!" %host
		sendmail(subject, text)
		sys.exit()
	# Wait for riak_kv service to start
	ssh1(host, user, ssh_key, 'riak-admin wait-for-service riak_kv')

def check_primary_vnodes(host, user, ssh_key):
	command = "riak-admin transfers"
	
	while True:
		output = ssh1(host, user, ssh_key, command)
		output = output.replace('\n', ' ').replace('\r', '')
		p = re.compile('does not have \d+ primary partitions running')
		m = p.match(output)
		if m:
			time.sleep(15)
		else:
			break

def check_handoffs(host, user, ssh_key):
	command  = "riak-admin transfers"
		
	while True:
			output = ssh1(host, user,ssh_key, command)
			output = output.replace('\n', ' ').replace('\r', '')
			if 'waiting to handoff' in output:
				time.sleep(15)
			else:
				break
	

try:

	for instance, node in riak_nodes.iteritems():
		instance_id = instance
		host = node
		user = 'EC2InstanceUserName'
		ssh_key = 'riak-test-key-pair.pem'
		# stop riak services
		stop_riak(host, user, ssh_key)
		# take snapshot		
		ebs_snapshot(instance_id)
		# start riak services and wait for kv service to come online
		start_riak(host, user, ssh_key)
		#check if any primary vnodes are down and wait till they are up
		check_primary_vnodes(host, user, ssh_key)
		# wait till handoffs are done
		check_handoffs(host, user, ssh_key)
	sendmail("Alert! Riak Backup Status", "Riak Backup script has executed successfully!")
except Exception as e:
	print "Exception :",str(e)

{% endhighlight %}

