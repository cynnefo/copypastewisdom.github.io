---
layout: post
title: "Restore All SQL Server Backups From a Directory"
excerpt: ""
modified: 2014-05-11 20:16:05 +0530
categories: articles
tags: [AWS, SQLServer]
comments: true
share: true
---

I had to setup database mirroring for 50 databases on amazon EC2 servers which are in different availability zones. Mirroring needs to be initiated with a full and at least one transaction log restores. Restoring 50 databases manually is a cumbersome task. Some of these databases have multiple data files and the logical names are different. I initially tried to generate restore scripts using `TSQL WHILE or CURSOR` but I quickly lost the plot with multiple temp tables and variables. You need to first run a `RESTORE HEADERONLY` for each database file, get the database name, run `RESTORE VERIFYONLY` for Logical/physical file names, build restore scripts. 

Python seemed perfect tool for the job because you only need to run two commands on SQL Server and the rest of it is manipulating file names, looping through logical files etc. This is a perfect excuse to try Python at work :). I used [pyodbc](https://code.google.com/p/pyodbc/) to make connection to database. If you dont have `pyodbc` module, you might want to install it before trying the script. The script can handle databases with multiple data files. Default target data file/logfile directories can be declared.


Here is the script:


{% highlight python %}
import os, sys
import pyodbc

backupdir = 'F:\\Backups\\'
conn = pyodbc.connect('DRIVER={SQL Server};SERVER=localhost;DATABASE=DBAdmin;UID=dbadmin;PWD=sysadmin')
cursor = conn.cursor()
DataDir = 'D:\\SQLData\\Data'
LogDir = 'E:\\SQLData\\Log'


def iter_islast(row):
    it = iter(row)
    prev = it.next()
    for item in it:
        yield prev, False
        prev = item
    yield prev, True


for root, dirs, files in os.walk(backupdir, topdown = False):
    for name in files:
        TempSQL = ""
        f = os.path.join(root, name)
        print "--Restore Script for backupfile " + f + " below: "
        print "--==============================================="
        
        cursor.execute("restore headeronly from disk = '%s';" % f)
        
        rows = cursor.fetchall()
        name = ""
        for row in rows:
            
            TempSQL = "RESTORE DATABASE  " + row.DatabaseName + " FROM DISK = '" + f + "'  WITH \n"
        
        cursor.execute("restore filelistonly from disk = '%s';" % f)
        rows2 = cursor.fetchall()
         
        for row, islast in iter_islast(rows2):
            if islast:
                if row.Type == 'L':
                    TempSQL += " MOVE " + "'"+row.LogicalName+"' TO  '" + LogDir + "\\" + row.LogicalName + ".ldf', NORECOVERY \n"
                else:
                    TempSQL += " MOVE " + "'"+row.LogicalName+"' TO  '" + DataDir + "\\" + row.LogicalName + ".mdf', NORECOVERY \n"
            else:
                if row.Type == 'L':
                    TempSQL += " MOVE " + "'"+row.LogicalName+"' TO  '" + LogDir + "\\" + row.LogicalName + ".ldf', \n"
                else:
                    TempSQL += " MOVE " + "'"+row.LogicalName+"' TO  '" + DataDir + "\\" + row.LogicalName + ".mdf', \n"
                
        print TempSQL
{% endhighlight %}

Enjoy!