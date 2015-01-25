---
layout: post
title: "MongoDB Notes: Memory Model"
excerpt: "Understanding MongoDB memory"
modified: 2014-05-11 20:16:05 +0530
categories: articles
tags: [MongoDB]
comments: true
share: true
---

These are notes I took from M202 class and from reading [docs](http://docs.mongodb.org/manual/faq/storage/). I intend to keep publishing notes here throughout the course, because you know writing is a process of thinking with clarity. 
{: .notice}
 
MongoDB uses mmap() (memory mapped files). If you have a data worth 500 GB and your server has 32 gigs or RAM, we can't fit this whole data on in physical memory. So the operating system maps the file to a region in virtual memory using mmap. When a `Mongod` process requests parrticular data, mmap will then fetch that data from the disk to physical RAM (accessing lazily, when needed). Instead of writing their own memory management solution, Mongo guys tapped into Kernel capabilities. Think about buffer manager in SQL Server. It does not read pages directly from disk to RAM; instead, it makes a request to the buffer manager which fetches the page from buffer cahce (Thats why you keep an eye on things like buffer cache hit ratio and page life expectancy)

That sounds easy and interesting but we will soon run into a problem. When you start loading pages into RAM from disk when needed, you will soon run into a situation where your physical memory is full. What happens then? You need to discard some pages to accommodate new ones, right? How does kernel decide which page to discard? Enter LRU algorithm ([Least Recently Used](http://en.wikipedia.org/wiki/Cache_algorithms)). Lets say the very first piece of data or page read into memory is accessed often (may be index related data), then this will not be the first to discarded. Kernel looks for the least recently used and starts discarding them(or persists to disk, if it has to).


# What is workingset? 

The part of data that is used most often by MongoDB is the workingset (like indexes that are frequently scanned). The subset of the data that you use reqularly is your workinset. How to find what is your current workingset? on 2.6, when you run `db.serverStatus({workingSet: 1})` you'll see the current working set (workingSet is case sensitive. I spent quite a lot of time trying to figureout why wouldn't my server show me my current workingset even after importing a collection of hundred thousand records). On my test server, it looks like below:

{% highlight json %}

        "workingSet" : {
                "note" : "thisIsAnEstimate",
                "pagesInMemory" : 1235,
                "computationTimeMicros" : 13898,
                "overSeconds" : 20546
         },
{% endhighlight %}

The default page size is 4k and I have 1235 pages in memory i.e close to 5 megs. You'll also notice something called the Resident Memory in serverStatus. The Workingset will always be a subset of resident memory. On a really busy system with a lot of activity, the resident memory and workingset can be quite close. This can be an indication that the server has memory pressure. 

{% highlight json %}
"writeBacksQueued" : false,
        "mem" : {
                "bits" : 64,
                "resident" : 45,
                "virtual" : 779,
                "supported" : true,
                "mapped" : 240,
                "mappedWithJournal" : 480
        },
{% endhighlight %}

So what this means is you get the best performance if you can fit your entire data in RAM(obviously!), or if you can fit the entire resident memory on your RAM and finally - atleast if you can fit the most frequently accessed indices in RAM - in that order.
{: .notice}

But its not all hunky dory. Resident Memory can be unreliable at times. Will the resident memory ever be lower than your workinset? Yes. Enter **Journaling**.

# What is journaling? 

Journaling is that which takes care of durability function in MongoDB. Think transaction logs in SQL Server (binary logs in MySQL, if you prefer). In a simple setup where journaling is not involved, the following happens

* The data file is mapped to a `shared view` in virtual memory using *mmap* call. 
* The `mongod` process will make requests and make any changes to what is called a `shared(public) view` in virtual memory
* The chnages are persisted to the disk every 60 sec. 

See, we have a problem. 60 sec worth of data loss is unacceptable to even those who serve 'yo dawg' memes. This is exactly the problem that Journaling solves. How does it solve it? 

* With journaling - there is another copy of the `shared view` called `private view` in memory
* The `mongod` process, instead of hitting the public view directly - makes changes to the `private view` first and  these changes are synced to a journal file every 100 ms. 
* When a crash happens (and they do happen), the journal file can play back the changes (I was almost tempted to say 'transactions', oh well - Mongo doesn't do transactions, Yo!)

This is similar to how transaction log works on SQL Server. The transactions are written to the transaction log first and there is a checkpoint running in the backend to persis committed transactions to the disk. 

So turning on journaling essentially means doubling the size of virtual memory. Also, journaling comes into play only when you do the writes. If you can get away with 100% reads all the time, you can do away with journaling(good luck finding a business like that)

So how does all this make resident memory be lower than workingset? If you have a read heavy system with 5% writes - then there will be a lot of paging out of data and this will make it look like you are not using all of your available RAM. It is also an unfortunate side-effect of remapping to file system cache that happens as part of journaling. 






