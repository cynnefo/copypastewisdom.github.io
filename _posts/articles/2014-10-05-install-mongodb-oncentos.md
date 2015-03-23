---
layout: post
title: "Installing MongoDB on CentOS"
categories: articles
excerpt: "Install MongoDB on CentOS"
tags: [mongodb]
date: 2014-10-05T13:56:53+05:30
comments: true
share: true
---

Create a Repo File

{% highlight bash %}
touch /etc/yum.repos.d/mongodb.repo
{% endhighlight %}

And append the following to the file.

{% highlight yaml %}
[mongodb]
name=MongoDB Repository
baseurl=http://downloads-distro.mongodb.org/repo/redhat/os/x86_64/
gpgcheck=0
enabled=1
{% endhighlight %}

The base URL in the above will change to `http://downloads-distro.mongodb.org/repo/redhat/os/i686/` if you need to install Mongo on 32 bit. But then, why would you run Mongo on 32 bit. It is supposed to process humongous amounts of data and 32 bit will be bad choice.

now, go ahed and run this -

{% highlight bash %}
sudo yum install -y mongodb-org
{% endhighlight %}

Start mongodb

{% highlight bash %}
sudo service mongod start
{% endhighlight %}

And boom! you are up and running. To verify whether it is has successfully started, you may check the mongodb log

{% highlight bash %}
cat /var/log/mongodb/mongod.log | grep 'waiting for connections on'
{% endhighlight %}

You should see this line: `[initandlisten] waiting for connections on port 27017`

Also, it may be a good idea to enable service at the startup.

{% highlight bash %}
sudo chkconfig mongod on
# stop and restart
sudo service mongod stop
sudo service mongod restart
{% endhighlight %}

Enjoy!
