# InfluxDB1
## inspecting 1.x data  ( Command Line )
connect to database using client -
```
[puneets@server1]~> influx
Connected to http://localhost:8086 version 1.8.10
InfluxDB shell version: 1.8.10
>
```
check databases
```
> show databases;
name: databases
name
----
_internal
monitoring
test
>
```

select database and check the "measurements"
```
>use monitoring
> show measurements;
name: measurements
name
----
ALERTS
ALERTS_FOR_STATE
Cluster_PSUSP
Cluster_SSUSP
Cluster_UNKNOWN
Cluster_USUSP
NJOBS
Pending_Jobs
Running_Jobs
...
up
```

check the fields in a measurement - 
```
> SHOW FIELD KEYS FROM "queue_running"
name: queue_running
fieldKey fieldType
-------- ---------
value    float
```
check tags in specific measurement - 
```
> SHOW TAG KEYS FROM "queue_running"
name: queue_running
tagKey
------
Total_queue_running
__name__
instance
job
>
```

check first 20 records
```


> SELECT * FROM "queue_running" LIMIT 20
name: queue_running
time                Total_queue_running         __name__      instance       job        value
----                -------------------         --------      --------       ---        -----
1674024512157000000 Running jobs in Queue A     queue_running localhost:933 wen-cluster 41
1674024512157000000 Running jobs in Queue B     queue_running localhost:933 wen-cluster 122
1674024512157000000 Running jobs in Queue C     queue_running localhost:933 wen-cluster 117
1674024512157000000 Running jobs in Queue D     queue_running localhost:933 wen-cluster 0
1674024512157000000 Running jobs in Queue E     queue_running localhost:933 wen-cluster 0
1674024512157000000 Running jobs in Queue F     queue_running localhost:933 wen-cluster 0
1674024512157000000 Running jobs in Queue G     queue_running localhost:933 wen-cluster 0
1674024512157000000 Running jobs in Queue H     queue_running localhost:933 wen-cluster 1
1674024512157000000 Running jobs in Queue I     queue_running localhost:933 wen-cluster 211
1674024512157000000 Running jobs in Queue J     queue_running localhost:933 wen-cluster 0
1674024512157000000 Running jobs in Queue K     queue_running localhost:933 wen-cluster 0
1674024512157000000 Running jobs in Queue L     queue_running localhost:933 wen-cluster 89
1674024512157000000 Running jobs in Queue M     queue_running localhost:933 wen-cluster 1454
1674024512157000000 Running jobs in Queue N     queue_running localhost:933 wen-cluster 2371
1674024512157000000 Running jobs in Queue O     queue_running localhost:933 wen-cluster 264
1674024512157000000 Running jobs in Queue P     queue_running localhost:933 wen-cluster 0
1674024512157000000 Running jobs in Queue Q     queue_running localhost:933 wen-cluster 0
1674024512157000000 Running jobs in Queue R     queue_running localhost:933 wen-cluster 60
1674024572158000000 Running jobs in Queue A     queue_running localhost:933 wen-cluster 41
1674024572158000000 Running jobs in Queue B     queue_running localhost:933 wen-cluster 122
```
to see time in human radable format use following before running select - 
```
>precision rfc3339
```

## Inspecting Data ( Python Client )

create a virtual env
```
python3 -m virtualenv influx1.x
source bin/activate
pip3 install influxdb
```
now create a python code which performs following in order - 
1. connect to 1.x database instance
2. list databases
3. list measurements
4. for monitoring measurement  , print TAG and FIELD data
5. for monitoring measurement, get the oldest record available

```
#!/usr/bin/env python3
from influxdb import InfluxDBClient
client1 = InfluxDBClient(host='den-grafana-01', port=8086)
_dbs=client1.get_list_database()

print("DATABASES:",_dbs)
client1.switch_database('monitoring')
measurements = client1.query('SHOW MEASUREMENTS')


for _measurement in measurements.get_points():
    print("MEASUREMENT:",_measurement['name'])
    _measurement=_measurement['name']
    _tags=client1.query('SHOW TAG KEYS FROM {_measurement}'.format(_measurement=_measurement)).get_points()
    _fields=client1.query('SHOW FIELD KEYS FROM {_measurement}'.format(_measurement=_measurement)).get_points()
    print("TAGS:",[_tag['tagKey'] for _tag in _tags])
    print("FIELDS:",[_field['fieldKey'] for _field in _fields])


    result = client1.query(f"SELECT * FROM {_measurement} ORDER BY time ASC LIMIT 1".format(_measurement=_measurement))

    # extract the oldest data point from the query result
    oldest_data = list(result.get_points())[0]
    print("DATA:",oldest_data)
```
Here is the output - 
```
DATABASES: [{'name': '_internal'}, {'name': 'monitoring'}, {'name': 'test'}]

MEASUREMENT: ALERTS
TAGS: ['__name__', 'alertname', 'alertstate', 'instance', 'job', 'severity']
FIELDS: ['value']
DATA: {'time': '2023-02-10T15:19:41.456000Z', '__name__': 'ALERTS', 'alertname': 'MonitoringInstanceDown', 'alertstate': 'firing', 'instance': '100.73.7.11:933', 'job': 'lsf_jobs', 'severity': 'critical', 'value': 1.0}
MEASUREMENT: ALERTS_FOR_STATE
TAGS: ['__name__', 'alertname', 'instance', 'job', 'severity']
FIELDS: ['value']
DATA: {'time': '2023-02-10T15:19:41.456000Z', '__name__': 'ALERTS_FOR_STATE', 'alertname': 'MonitoringInstanceDown', 'instance': '100.73.7.11:933', 'job': 'lsf_jobs', 'severity': 'critical', 'value': 1676042261.0}
MEASUREMENT: Cluster_PSUSP
TAGS: ['PSUSP', '__name__', 'instance', 'job']
FIELDS: ['value']
DATA: {'time': '2023-01-18T06:48:32.157000Z', 'PSUSP': 'Cluster PSUSP JOBS', '__name__': 'Cluster_PSUSP', 'instance': 'localhost:933', 'job': 'wen-cluster', 'value': 8.0}
MEASUREMENT: Cluster_SSUSP
TAGS: ['SSUSP', '__name__', 'instance', 'job']
FIELDS: ['value']
```


# Migrating Data ( 1.x to 2.x )
## convert the 1.x data into line protocol format using influx_inspect
from the inspection and config file analysis, we know that in source 1.x database we have - 
1. database "monitoring"
2. 
```

[meta]
  dir = "/database/influxdb/meta"

[data]
  wal-dir = "/database/influxdb/wal"
  dir = "/database/influxdb/data"
```

we will now attempt migration of 1 hour data under - /database/data_migrate/try1 
```
time influx_inspect export -database "monitoring" -datadir "/database/influxdb/data" -waldir "/database/influxdb/wal" -out "/database/data_migrate/try1/backup1" -start 2023-01-18T06:48:32.157000Z -end 2023-01-18T09:48:32.157000Z
```
expected  output   - 
```
writing out tsm file data for monitoring/autogen...complete.
writing out wal file data for monitoring/autogen...complete.
152.770u 26.230s 7:18.98 40.7%  0+0k 41416024+45358592io 369pf+0w
```
size of file - 
```
[1.x]/database/data_migrate/try1> ls -lrt backup1
-rw-r--r-- 1 monadmin monadmin 23223595239 Mar 19 04:09 backup1
[1.x]/database/data_migrate/try1>
```

>NOTE: 22G is big, and due to lack of space on the target system ( 2.x) we decided to use 1 hour data 

scp backup1 puneet@den-grafana-02:~/

## importing line protocol data into influx 2.x
notice the change in terminology  - 
database (1.x) = Bucket ( 2.x)

So create a database/Bucket in influx 2.x as -
```
2.x]~> /usr/local/bin/influx261/influx bucket create -n monitoring -o amperecomputing.com
ID                      Name            Retention       Shard group duration    Organization ID         Schema Type
e8bbc11afb5fb90c        monitoring      infinite        168h0m0s                8342dd7d8cd5e12a        implicit
2.x]~>
```

get your token 
```
2.x]~> /usr/local/bin/influx261/influx auth ls
ser Name        User ID                 Permissions
0ae9599f56c47000        puneet336's Token       QjTuIesU0Fm5oafWgP2uigtB1a77e7zytNbv07vdD_0qT4X9H6L0bb-L_ovfoPxFeYxAj6g== puneet336       0ae9599f41c47000        [read:/authorizations write:/authorizations read:/buckets write:/buckets read:/dashboards write:/dashboards read:/orgs write:/orgs read:/sources write:/sources read:/tasks write:/tasks read:/telegrafs write:/telegrafs read:/users write:/users read:/variables write:/variables read:/scrapers write:/scrapers read:/secrets write:/secrets read:/labels write:/labels read:/views write:/views read:/documents write:/documents read:/notificationRules write:/notificationRules read:/notificationEndpoints write:/notificationEndpoints read:/checks write:/checks read:/dbrp write:/dbrp read:/notebooks write:/notebooks read:/annotations write:/annotations read:/remotes write:/remotes read:/replications write:/replications]
```


```
2.x]~> /usr/local/bin/influx261/influx write --org amperecomputing.com --bucket monitoring --token QjTuIesU0Fm5oafWgP2uigtB1a77e7zytNbv07vdDj6q__06vIbeBe2i_0qT4X9H6L0bb-L_ovfoPxFeYxAj6g== --file ./backup1
```

expected output  was Error2. but despite the Error2, we were able to query the monitoring database and list the data using `influx v1 shell` command



Error1
```
Error: failed to write data: 400 Bad Request: unable to parse 'CREATE DATABASE monitoring WITH NAME autogen': invalid field format
```
Sloution - 
```
]~> grep -rin "# writing tsm data" ./backup1
7:# writing tsm data
]~> sed -i '1,7d' ./backup1 
```

Error2
```
Error: failed to write data: 400 Bad Request: unable to parse 'skipped missing file: /database/influxdb/wal/monitoring/autogen/100/_58045.wal': invalid field format
```


## inspecting 2.x data  ( Command Line )
```
2.x]~> /usr/local/bin/influx261/influx query --org amperecomputing.com --token "QjTuIesU0Fm5oafWgP2uigtB1a77e7zytNbv07vdDj6q__06vIbeBe2i_0qT4X9H6L0bb-L_ovfoPxFeYxAj6g==" 'from(bucket: "monitoring") |> range(start: 0) |> filter(fn: (r) => r._measurement == "queue_running") |> yield()'
Result: _result
Table: keys: [_start, _stop, Total_queue_running, __name__, _field, _measurement, instance, job]
                   _start:time                      _stop:time     Total_queue_running:string         __name__:string           _field:string     _measurement:string         instance:string              job:string                      _time:time                  _value:float
------------------------------  ------------------------------  -----------------------------  ----------------------  ----------------------  ----------------------  ----------------------  ----------------------  ------------------------------  ----------------------------
1970-01-01T00:00:00.000000000Z  2023-03-19T12:39:03.741057134Z  Running jobs in Queue RsvLong           queue_running                   value           queue_running          localhost:9337             den-cluster  2023-01-18T08:49:32.157000000Z                            41
1970-01-01T00:00:00.000000000Z  2023-03-19T12:39:03.741057134Z  Running jobs in Queue RsvLong           queue_running                   value           queue_running          localhost:9337             den-cluster  2023-01-18T08:50:32.157000000Z                            41
1970-01-01T00:00:00.000000000Z  2023-03-19T12:39:03.741057134Z  Running jobs in Queue RsvLong           queue_running                   value           queue_running          localhost:9337  
```


/usr/local/bin/influx261/influx query was hanging up for me so i used GUI

<img width="938" alt="image" src="https://user-images.githubusercontent.com/5935825/226174477-d150ade2-f6b7-48d5-87cc-27c618fda1ac.png">

to query buckets - 
<img width="791" alt="image" src="https://user-images.githubusercontent.com/5935825/226174527-ad376677-dab4-478b-ba2e-7728f7e349fe.png">







