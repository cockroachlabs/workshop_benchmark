# Benchmark Workshop

This workshop will have some demo's to show the technique for benchmarking CockroachDB.  It will build upon various built-in tools and be extended to show how to use Jmeter to easily code transactions and queries.  Jmeter introduces a weighted driving of workload transations/micro-services to show how to benchmark growth.

## Demo creation

```
roachprod create glenn-bench --gce-machine-type 'n1-standard-16' --nodes 9 --lifetime 132h
roachprod stage glenn-bench release v20.1.4
roachprod start glenn-bench
roachprod pgurl glenn-bench:1
roachprod adminurl glenn-bench:1
        http://glenn-bench-0001.roachprod.crdb.io:26258/

## -- configure driver machine
##
roachprod create glenn-drive -n 1 --lifetime 168h
roachprod stage glenn-drive release v20.1.4
roachprod ssh glenn-drive:1
sudo mv ./cockroach /usr/local/bin

sudo apt-get update -y
sudo apt-get install haproxy -y
cockroach gen haproxy --insecure   --host=10.142.0.26   --port=26257 
nohup haproxy -f haproxy.cfg &

## Test connectivity
cockroach sql --execute 'select 1' --insecure
```

### Adjusting Metrics parameters

```
SET CLUSTER SETTING diagnostics.sql_stat_reset.interval='12hr';
SET CLUSTER SETTING diagnostics.forced_sql_stat_reset.interval='24hr';
SET CLUSTER SETTING diagnostics.reporting.interval='12h';
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
cockroach workload run ycsb --workload F --concurrency 36

```

## Demo Round3  :: workload run querybench  :: Scaling Ingest
```
## Download workload binary if needed
wget https://edge-binaries.cockroachdb.com/cockroach/workload.LATEST
chmod 755 workload.LATEST
cp -i workload.LATEST /usr/local/bin/workload
chmod u+x /usr/local/bin/workload
```

Create simple schema to test logging function:
```sql
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
```


### Queries files ran by querybench

Query ran by `b1_query.sql`
```sql
insert into mylog (eventname) select 'b1' from generate_series(1,1);
```

Query ran by `b10_query.sql`
```sql
insert into mylog (eventname) select 'b10' from generate_series(1,10);
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
sudo apt-get update
#apt install openjdk-11-jre-headless
#apt install openjdk-8-jre-headless
#apt install openjdk-9-jre-headless
sudo apt install default-jre
wget https://apache.cs.utah.edu//jmeter/binaries/apache-jmeter-5.2.1.tgz
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


##  Student Labs

Will need to really create a separate cluster for each student to do full-scale testing.  For 