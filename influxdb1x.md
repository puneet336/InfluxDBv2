# InfluxDB1
## inspecting data  ( Command Line )
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
