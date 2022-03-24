# Benchmark Workshop

This workshop will have some demo's to show the technique for benchmarking CockroachDB.  It will build upon various built-in tools and be extended to show how to use Jmeter to easily code transactions and queries.  Jmeter introduces a weighted driving of workload transations/micro-services to show how to benchmark growth.

## Demo creation

```
roachprod create ${USER:0:6}-bench --gce-machine-type 'n1-standard-16' --nodes 9 --lifetime 48h
roachprod stage ${USER:0:6}-bench release v21.2.5
roachprod start ${USER:0:6}-bench
roachprod pgurl ${USER:0:6}-bench:1
roachprod adminurl ${USER:0:6}-bench:1

## -- configure driver machine
##
roachprod create ${USER:0:6}-drive -n 1 --lifetime 48h
roachprod stage ${USER:0:6}-drive release v21.2.5
roachprod ssh ${USER:0:6}-drive:1
sudo mv ./cockroach /usr/local/bin

sudo apt-get update -y
sudo apt-get install haproxy -y
cockroach gen haproxy --insecure   --host=10.142.0.3   --port=26257 
nohup haproxy -f haproxy.cfg &

## Test connectivity
cockroach sql --execute 'select 1' --insecure
```

### Adjusting Metrics parameters

```
SET CLUSTER SETTING diagnostics.sql_stat_reset.interval='12hr';
SET CLUSTER SETTING diagnostics.forced_sql_stat_reset.interval='24hr';
SET CLUSTER SETTING diagnostics.reporting.interval='12h';

show cluster setting kv.range_merge.queue_enabled;
show cluster setting kv.range_split.by_load_enabled;
show cluster setting kv.range_split.load_qps_threshold;

set cluster setting kv.range_merge.queue_enabled='false';
set cluster setting kv.range_split.by_load_enabled='true';
set cluster setting kv.range_split.load_qps_threshold=200;
```

## Demo Round1  :: cockroach workload run kv
```
cockroach workload init kv --zipfian
cockroach workload run kv --concurrency 16 --duration 120s --zipfian 
cockroach workload run kv --concurrency 16 --duration 120s --zipfian --read-percent 50 --display-every 5s
```

## Demo Round2 :: cockroach workload run ycsb

Run this and watch splitting of ranges for ycsb table:
```
cockroach workload init ycsb --workload F --drop --insert-count 1000000
cockroach workload run ycsb --workload F --concurrency 36 --duration 300s

```

## Demo Round3  :: workload run querybench  :: Scaling Ingest
```
## Download workload binary if needed
wget https://edge-binaries.cockroachdb.com/cockroach/workload.LATEST
chmod 755 workload.LATEST
sudo cp -i workload.LATEST /usr/local/bin/workload
sudo chmod u+x /usr/local/bin/workload
```

Create simple schema to test logging function:
```sql
create database mybench_b1;
use mybench_b1;

create table mylog_b1(
    id uuid DEFAULT gen_random_uuid() primary key,
    ts timestamp DEFAULT now(),
    eventname string
);

create database mybench_b10;
use mybench_b10;

create table mylog_b10 (
    id uuid DEFAULT gen_random_uuid() primary key,
    ts timestamp DEFAULT now(),
    eventname string
);
```


### Queries files ran by querybench

Query ran by `b1_query.sql`
```sql
insert into mylog_b1 (eventname) select 'b1' from generate_series(1,1);
```

Query ran by `b10_query.sql`
```sql
insert into mylog_b10 (eventname) select 'b10' from generate_series(1,10);
```

### Quick scale up Ingest to find > 10k rows/second

Run these scripts and watch the scaling and HW consumption.
```bash

workload run querybench --query-file b1_query.sql --db mybench_b1 --duration 120s --concurrency 9
workload run querybench --query-file b1_query.sql --db mybench_b1 --duration 120s --concurrency 18

workload run querybench --query-file b10_query.sql --db mybench_b10 --duration 120s --concurrency 2
workload run querybench --query-file b10_query.sql --db mybench_b10 --duration 120s --concurrency 4
```


## Jmeter Configuration Notes

Install Java and get jmeter binary:

```bash
#apt install openjdk-11-jre-headless
#apt install openjdk-8-jre-headless
#apt install openjdk-9-jre-headless
sudo apt-get update
sudo apt install default-jre
wget https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.4.3.tgz
```

Place the `postgresql` driver into the JMeter lib directory:

```bash
wget -O postgresql-42.2.11.jar https://jdbc.postgresql.org/download/postgresql-42.2.11.jar
```

Start JMeter in GUI mode:

```bash
/path/to/jmeter/bin/jmeter
```

## Jmeter Demo

Launch the Sample GUI mode:
```
cd ~/git/apache-jmeter-5.2.1/bin
./jmeter.sh
## crdb_jmeter_example1.jmx file
```

Notice there are various functions available to use within the code:
+ [jmeter functions](https://jmeter.apache.org/usermanual/functions.html)

### DEMO :: mylogger

```
create database mylogger;
use mylogger;

create table mylog1 (
    id uuid DEFAULT gen_random_uuid(),
    ts timestamp DEFAULT now()::timestamp,
    eventname string,
    thread int,
    thread_group string,
    notes string,
    primary key (ts, id)
);

insert into mylog1 (eventname, thread, thread_group, notes ) values ('active', 3, 'ingestGroup', 'ingest rampup');

-- Dash Board Query
--
with lastEvents as (
    select ts, eventname
    from mylog1
    limit 100
)
select eventname, count(*) 
from lastEvents
group by 1;

```

##  Student Labs

Will need to really create a separate cluster for each student to do full-scale testing. 

Cluster + Driver for each Student:
```
for c in 1 2 3 4 5
do
  roachprod create glenn-bcluster${c} -n 3 --lifetime 36h
  roachprod stage glenn-bcluster${c} release v20.1.4
  roachprod start glenn-bcluster${c} 

  roachprod create glenn-bdrive${c} -n 1 --lifetime 36h
  roachprod stage glenn-bdrive${c} release v20.1.4
done

## login to each cluster and add "bench" user and keys

for d in 1 2 3 4 5
do
  roachprod ssh glenn-bdrive${d}
done

sudo adduser -q --home /home/bench bench --disabled-password
sudo su - bench
mkdir .ssh
cd .ssh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCpbeqGNW+WasEoNUf3OvC+Di032Uxceuqhf7jwJ3bJ34a3icRMsukpoSySvYXIA+EZ752w4cuViyGNwZye71H+hEEd4q7N04KiuTE0cnHKkrN7aegUPmCRWh/tPRfaEOiE1KyI/qPtkn2T1cRRmYyVqimkxwLyEPTzJbjmN24WaktWkCTW+Hi2iifV8AqB+Kwb2qSvSd4RdBHwUYxHpPjulbpToRPpDA1bZmrqLgSsYmlALOPZA8WDS+B2/4xL5izQNd0IfEDzbmdDqzDrIGA4gvcpdu/9Smhq1YoEVXWWTA0bL9COkJUy6EAi5EiMfwA6hF1tg38vv06BIdKNKmV8b7C26d2d8VExXYOX+088F2QMLaN5SmryTIMz7/ZaAsc0053dpQzzKQdbKZnh3BAR3sp9qIHnj+Nho5WzANr25UKEgIeyVX99NM/zkahrtyguDgiR4OpxgP5yox2lj4YdxMu15NrjVnRBgxUw2AxR1iyIyFIIbhT6LRkEhshW4Tc= bench" > ./authorized_keys
chmod go-rw authorized_keys

for d in 1 2 3 4 5
do
  roachprod ssh glenn-bdrive${d}
done

sudo apt-get update -y
sudo apt-get install haproxy -y
./cockroach gen haproxy --insecure   --host=10.142.0.16   --port=26257 
./cockroach gen haproxy --insecure   --host=10.142.0.24   --port=26257 
./cockroach gen haproxy --insecure   --host=10.142.0.81   --port=26257 
./cockroach gen haproxy --insecure   --host=10.142.0.113   --port=26257 
./cockroach gen haproxy --insecure   --host=10.142.0.117   --port=26257 

nohup haproxy -f haproxy.cfg &
./cockroach sql --execute 'select 1' --insecure

for d in 1 2 3 4 5
do
  roachprod ssh glenn-bdrive${d}
done

wget https://edge-binaries.cockroachdb.com/cockroach/workload.LATEST
chmod 755 workload.LATEST
mv ./workload.LATEST ./workload
chmod u+x ./workload

echo "insert into mylog_b1 (eventname) select 'b1' from generate_series(1,1);" > b1_query.sql
echo "insert into mylog_b10 (eventname) select 'b10' from generate_series(1,10);" > b10_query.sql

for d in 1 2 3 4 5
do
  roachprod sql glenn-bdrive${d}
done

create database mybench_b1;
use mybench_b1;

create table mylog_b1 (
    id uuid DEFAULT gen_random_uuid() primary key,
    ts timestamp DEFAULT now(),
    eventname string
);

create database mybench_b10;
use mybench_b10;

create table mylog_b10 (
    id uuid DEFAULT gen_random_uuid() primary key,
    ts timestamp DEFAULT now(),
    eventname string
);

create database mylogger;
use mylogger;

create table mylog1 (
    id uuid DEFAULT gen_random_uuid(),
    ts timestamp DEFAULT now()::timestamp,
    eventname string,
    thread int,
    thread_group string,
    notes string,
    primary key (ts, id)
);


./workload run querybench --query-file b1_query.sql --db mybench_b1 --duration 120s --concurrency 9
./workload run querybench --query-file b1_query.sql --db mybench_b1 --duration 120s --concurrency 18
./workload run querybench --query-file b10_query.sql --db mybench_b10 --duration 120s --concurrency 2
./workload run querybench --query-file b10_query.sql --db mybench_b10 --duration 120s --concurrency 4

for d in 1 2 3 4 5 
do 
  #roachprod ssh glenn-bcluster${d}:1 "./cockroach sql --insecure --execute 'drop database if exists tpcc${d} cascade'"
  roachprod ssh glenn-bcluster${d}:1 "./cockroach sql --insecure --execute \"restore database tpcc from 'gs://querylabs/backup_lab1'\" "
done 

```

