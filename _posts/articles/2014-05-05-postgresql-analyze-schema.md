---
layout: post
title: "PostgreSQL - Analyze specific schema - Shell script"
categories: articles
excerpt: "updating statistics of a schema in multi tenant database"
tags: [PostgreSQL, Databases]
comments: true
share: true
---

[Analyze](http://www.postgresql.org/docs/9.1/static/sql-analyze.html) in PostgreSQL is like `sp_updatestats` of *SQL Server*. Analyze gathers statistics about number of rows in a table and stores them in pg_statistics system catalog, which are used by query planner to create optimal query plans. Below is a shell script that is useful to update statistics on a schema. If you are on a single database and multi-tenant model, this is pretty useful to update the target tenant schema, without updating the entire database. 

Before executing this, you might want to add postgres user, password and host to your [pgpass](http://www.postgresql.org/docs/9.1/static/libpq-pgpass.html) file, so that you can execute this shell script without being prompted for password. 

{% highlight bash %}
vi /root/.pgpass 
 
#Add details to file hostname:port:database:username:password
*:*:*:dbadmin:SysAdminPwd
 
#set file permissions
chmod 600 /root/.pgpass
{% endhighlight %}

Now the shell script: You can download script from [this link](https://www.dropbox.com/s/d00rfkkxzmyrvbl/analyze_schema.sh)

{% highlight bash %}
#!/bin/sh 
# 
# 
# Title    : analyze_schema.sh 
# Description:   
#  Shell script  
#   
# Options: 
#  name of the target schema 
# 
# Requirements: 
#  psql installed and added path to binary in PATH system variable.  
#   
# Examples: 
# $HOME/analyze_schema.sh "MySchemaName" 
# 
  
SCHEMANAME="$1"
DBNAME="${2:-MyDatabase}"
DBUSER="${3:-dbadmin}"
DBPORT="${4:-7432}"
   
ANALYZESQL="anaylze_schema.sql"
ANALYZELOG="analyze_schema.log"
  
test=$(psql -A -1 -t -U $DBUSER -d $DBNAME -p $DBPORT -c "select * from pg_namespace where nspname='$SCHEMANAME'") 
if [[ "$test" == "" ]]; then
  echo "Schema \"$SCHEMANAME\" in db \"$DBNAME\" for dbuser \"$DBUSER\" and port \"$DBPORT\" does not exists"
  exit 1 
fi
  
GENANALYZESQL=" 
select 'ANALYZE VERBOSE \"'||current_database()||'\".\"'||t.schemaname||'\".\"'||t.tablename||'\"; ' as analyze_statement 
from pg_tables t 
where schemaname = '$SCHEMANAME'
order by t.tablename; 
" 
  
echo "Started generating analyze script for Schema \"$SCHEMANAME\" started at $(date +'%Y-%m-%d %T')"
psql -A -q -1 -t -p 7432 -U dbadmin -d MyDatabase -c "$GENANALYZESQL" > "$ANALYZESQL" 2>"$ANALYZELOG"
if ! [[ "$(cat $ANALYZELOG)" == "" ]]; then
  echo "Generation of analyze script for schema \"$SCHEMANAME\" in db \"$DBNAME\" for dbuser \"$DBUSER\" and port \"$DBPORT\" had error(s):"
  cat "$ANALYZELOG"
  exit 1 
fi
echo "Analyzing schema \"$SCHEMANAME\" started at $(date +'%Y-%m-%d %T')"
psql -A -q -1 -t -p 7432 -U dbadmin -d MyDatabase -f "$ANALYZESQL" 2>&1 | /usr/sbin/prepend_datetime.sh >> "$ANALYZELOG"
echo "Analyzing schema \"$SCHEMANAME\" ended at $(date +'%Y-%m-%d %T')"
echo "Analyzing schema sql script is in file $ANALYZESQL. Analyze log is below:"
cat "$ANALYZELOG"
{% endhighlight %}
