# InfluxDB1
## inspecting data 
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


