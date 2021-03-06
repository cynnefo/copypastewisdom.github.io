---
layout: post
title: Amazon RDS Encryption and Benchmarking PostgreSQL
excerpt: Benchmarking Amazon encrypted RDS with PGBench
categories: articles
tags:
- AWS
- PostgreSQL
comments: true
share: true
date: '2015-01-24T00:00:00.000+00:00'
---

This post was originally written for [Minjar](http://minjar.com/) Blog and it was first published [here](http://blog.minjar.com/post/108724853340/rds-encryption-and-benchmarking-postgresql). Go check out that blog, lots of interesting stuff in there :)
{: .notice}

Amazon has recently [announced](https://aws.amazon.com/blogs/aws/new-encryption-options-for-amazon-rds/) new **data at rest** encryption option for PostgreSQL, MySQL and Oracle. While Enterprise editions of Microsoft SQL Server and Oracle already had encryption in the form of AWS Managed Keys, this new feature extends support to PostgreSQL and MySQL. 

We took the new encryption enabled Postgres instance out for a spin and this is what we learned:  

* An existing unencrypted RDS instance can not be converted to encrypted one.
* Restoring a backup/snapshot of an unencrypted RDS into an encrypted RDS is not supported.  
* Once you provision an encrypted RDS instance, there is no going back. You cant change it to unencrypted. 
* You can not have unencrypted read replicas for an encrypted RDS or vice versa.  
* Not all instance classes support encryption. Check the list of instance classes that support encryption [here](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html). 
 

What all of the above means is - if you are trying to upgrade an existing RDS to enable encryption, its not *that* straight forward. So the only way left is to take a logical backup(export data), spin a new RDS instance with encryption enabled and import data from logical backup. 

## How about the performance? 
			
Before suggesting our customers to go ahead with encryption, we wanted to test the effect of enabling encryption in terms of performance. We launched a couple of PostgreSQL RDS instances in Tokyo region. One of them encrypted and the other is unencrypted. Both these instances are of the same instance class - `db.m3.medium`. 

We ran a simple insert test to insert 10 million rows on both the instances using the following query

{% highlight sql %}

# Create Table 
CREATE TABLE names (id INT, NAME VARCHAR(100)); 

#Run the Insert 
INSERT INTO names (
	id
	,NAME
	)
SELECT x.id
	,'my name is #' || x.id
FROM generate_series(1, 10000000) AS x(id);
{% endhighlight %}

The results were interesting. Encrypted RDS took `1 min 31 sec` and unencrypted RDS took `1 min 03 sec`. 

Another insert test with a slightly different query  
{% highlight sql %}
CREATE TABLE t_random01 AS

SELECT s
	,md5(random()::TEXT)
FROM generate_Series(1, 10000000) s;
{% endhighlight %}

Encrypted RDS took `2 min 10 sec` and unencrypted took  `1 min 40 sec.`

We ran the above tests multiple times with slightly different timelines but in each test, the encrypted RDS took more time to insert than the unencrypted one. We then decided run some benchmarking tests for read/write performance using [PGBench](http://www.postgresql.org/docs/devel/static/pgbench.html)

PGBench runs same set of SQL commands multiple times in same sequence and calculates transactions per second and it tends to give more accurate overall results. While pgbench allows you to run wide variety of tests like connection contentions, prepared and adhoc queries - we limited our tests to read/write related ones in this case.  

## Initializing PGBench

Assuming that you already have PGBench binaries installed, cd to that directory and initialize a database on both instances.

{% highlight bash %}
pgbench -U minjar -h hostname.rds.amazonaws.com -p 5432 -i -s 70 BenchMe
{% endhighlight %}

You may have to create a blank database BenchMe on your RDS instances. The above command will create `pgbench_accounts`, `pgbench_branches`, `pgbench_history` and `pgbench_tellers` tables and populates them with data. 

## Read Write Test

{% highlight bash %}
pgbench -U minjar -h hostname.rds.amazonaws.com -p 5432 -c 4 -j 2 -T 600 BenchMe
{% endhighlight %} 

The above command runs simple read/write workload on BenchMe database for 600 sec. Results below:

*Unencrypted RDS*

<figure>
	<img src="/images/1.png" alt="image">
</figure>

*Encrypted RDS*
<figure>
	<img src="/images/2.png" alt="image">
</figure>

The encrypted RDS is processing `0.187146` transactions per second which is slightly better than unencrypted `0.132844`. This is interesting. This better read/write performance of encrypted RDS is in spite of marginally slower writes compared to unencrypted. What gives? We shall see. 

## Read Only Test

{% highlight bash %}
pgbench -U minjar -h hostname.rds.amazonaws.com -c 4 -j 2 -T 600 -S BenchMe
{% endhighlight %} 

In the above command, `-S` switch makes sure that the workload is read-only and the time frame is 600 seconds. The results are below. 


*Unencrypted RDS*

<figure>
	<img src="/images/3.png" alt="image">
</figure>

*Encrypted RDS*

<figure>
	<img src="/images/4.png" alt="image">
</figure>

In this test too, encrypted RDS wins with higher number of transactions per second compared to encrypted RDS. We think Amazon provisions these encrypted instances on better disk subsystem and they gain considerably in terms of read performance. This is one of the reasons why the read/write test above went in favour of encrypted RDS. 

## Simple Write Test

{% highlight bash %}
pgbench -U minjar -h hostname.rds.amazonaws.com -c 4 -j 2 -T 600 -N BenchMe
{% endhighlight %} 

*Unencrypted RDS*

<figure>
	<img src="/images/5.png" alt="image">
</figure>

*Encrypted RDS*

<figure>
	<img src="/images/6.png" alt="image">
</figure>

As expected, unencrypted RDS wins this one in simple write test with `4.635805` transactions. This is in line with the initial findings from running ad-hoc inserts. 

## Summary of results

| **Test Description**|**Encrypted RDS**|**Unencrypted RDS** |
|Ad-hoc Insert 1 (10 million rows)|1 min 31 sec|**1 min 03 sec**|
|Ad-hoc Insert 2 (10 million rows)|2 min 10 sec|**1 min 40 sec**|
|Simple Write Test (pgbench)|**4.274140 transactions/sec**|4.635805 transactions/sec|
|Readonly Test(pgbench)|**20.869002 transactions/sec**|20.701378 transactions/sec|
|Read-Write Test(pgbench)|**0.187146 transactions/sec**|0.132844 transactions/sec

So the final verdict is - there will be a very slight performance hit in terms of writes when you enable RDS encryption. But Amazon more than makes up for it in terms of overall better performance. We suggest enabling RDS encryption, the slight write performance overhead is worth the security that encryption brings. 


