# Install and configure InfluxDB v2.6.1 on CentOS 7

## Setup influxdb service account
For running influxdb as non-root user, we need to create a `myinfluxdb` user and group. To create a new system user and group, you can use these two commands:
```
sudo groupadd -r myinfluxdb
sudo useradd -g myinfluxdb -d /var/lib/myinfluxdb -s /sbin/nologin --system myinfluxdb
```

## Download and install InfluxDB binaries - 
```
wget https://dl.influxdata.com/influxdb/releases/influxdb2-2.6.1-linux-amd64.tar.gz
tar xvfz influxdb2-2.6.1-linux-amd64.tar.gz
```

contents of tarball -
 
ls influxdb2_linux_amd64
```
influxd  LICENSE  README.md
```

create influxDB binary directory and place binary -
```
sudo mkdir -p /usr/local/bin/influx261/
sudo cp influxdb2_linux_amd64/influxd /usr/local/bin/influx261/
sudo chown root:root /usr/local/bin/influx261/influxd
```

## Configure Influxdb

Create directories for VictoriaMetrics metadata, data , wal and set ownership to myinfluxdb account as :
```
sudo mkdir -p /var/lib/myinfluxdb/meta
sudo mkdir -p /var/lib/myinfluxdb/data
sudo mkdir -p /var/lib/myinfluxdb/wal
chown -R myinfluxdb:myinfluxdb /var/lib/myinfluxdb/
```

setup logging directory 
```
sudo mkdir -p /var/log/myinfluxdb/
sudo chown myinfluxdb:myinfluxdb  /var/log/myinfluxdb/
```

Create new systemd services.
```
sudo vi /etc/systemd/system/myinfluxdb.service
```

influxd sends outputs all logs to console so we will take help of systemd service and rsyslog to manage logging. 
Add the following lines to the file:

[Unit]
Description=High-performance, cost-effective and scalable time series database, long-term remote storage for Prometheus
After=network.target

```
[Service]
Type=simple
User=myinfluxdb
Group=myinfluxdb
StartLimitBurst=5
StartLimitInterval=0
Restart=on-failure
RestartSec=1
Environment="INFLUXD_CONFIG_PATH=/usr/local/bin/influx261/influxd.yaml"

StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=myinfluxdbv26

ExecStart=/usr/local/bin/influx261/influxd run
ExecStop=/bin/kill -s SIGTERM $MAINPID

LimitNOFILE=65536
LimitNPROC=32000

[Install]
WantedBy=multi-user.target
```
since we have choses syslog for logging, create following configuration file at /etc/rsyslog.d/myinfluxdbv26.conf with - 
```
if $programname == 'myinfluxdbv26' then /var/log/myinfluxdb/output.log
& stop
```

create a configuration file /usr/local/bin/influx261/influxd.yml as - 
```
http-bind-address: "0.0.0.0:9099"
engine-path: /var/lib/myinfluxdb/engine
bolt-path: /var/lib/myinfluxdb/myinfluxd.bolt
sqlite-path: /var/lib/myinfluxdb/myinfluxd.sqlite
```

Configuration files are in order, now start / restart services - 
```
sudo systemctl daemon-reload
sudo systemctl restart rsyslog
sudo systemctl start myinfluxdb
```


Enable and start Influx service to run on system boot automatically.

```
$ sudo systemctl enable myinfluxdb.service --now
```



## Set up InfluxDB

Download and install InfluxDB client binaries - 
```
 wget https://dl.influxdata.com/influxdb/releases/influxdb2-client-2.6.1-linux-amd64.tar.gz
 tar -xf influxdb2-client-2.6.1-linux-amd64.tar.gz
 sudo cp influxdb2-client-2.6.1-linux-amd64/influx /usr/local/bin/influx261/
```

Now as we are using 9099 port for server, we need to update the default client configuration 
```
]> /usr/local/bin/influx261/influx config create --config-name myinfluxv2 --host-url http://localhost:9099 -a
Active  Name            URL                     Org
*       myinfluxv2      http://localhost:9099
```

now we need to create a default organization, user, bucket, and Operator API token. this can be done through UI (localhost:<port>) and CLI.
We use CLI method as - 
```
/usr/local/bin/influx261/influx setup --host "http://localhost:9099"
> Welcome to InfluxDB 2.0!
? Please type your primary username puneet336
? Please type your password *********
? Please type your password again *********
? Please type your primary organization name example.com
? Please type your primary bucket name monitoring_prom
? Please type your retention period in hours, or 0 for infinite 0
? Setup with these parameters?
  Username:          puneet336
  Organization:      example.com
  Bucket:            monitoring_prom
  Retention Period:  infinite
 Yes
User            Organization            Bucket
puneet336       amperecomputing.com     monitoring_prom
]>
```
 

> retention period: Enter nothing for an infinite retention period. other units are nanoseconds (ns), microseconds (us or µs), milliseconds (ms), seconds (s), minutes (m), hours (h), days (d), and weeks (w). The InfluxDB retention enforcement service is responsible for verifying if data stored in a bucket exceeds the defined retention period and removing it accordingly. This process is automated and eliminates the need for manual intervention, ensuring that outdated data is removed to optimize disk usage. The retention enforcement service is set to run every 30 minutes by default, but you can modify this interval using the storage-retention-check-interval configuration option.


To continue to use InfluxDB via the CLI, you need the API token created during setup. To view the token, log into the UI (http://localhost:9099) with the credentials created above. 


## Why migrate to influxDB2 
In version 1.7.x, you had four different components that one would assemble to form the TICK stack (Telegraf, InfluxDB, Chronograf and Kapacitor).
Query langiage is InfluxQL ( like SQL)
In version 2.0, InfluxDB becomes a single platform for querying, visualization and data manipulation.
Query Language is Flux, PromQL support ?




## Changes in InfluxDB

How to configure - 

Storage Dir(1.x), 
Here there are 3 components , namely - meta data , actual storage  and wal. these 3 paths can be configured - 

using config file:
[meta]
  dir = "/var/lib/myinfluxdb/meta"
[storage]
  dir = "/var/lib/myinfluxdb/data"
  wal-dir = "/var/lib/myinfluxdb/wal"


Storage Dir(2.x) :


Here, there are 2 components - bolt and engine. 
engine includes - Write Ahead Log (WAL),Cache,Time-Structed Merge Tree (TSM), Time Series Index (TSI)


default: ~/.influxdbv2/influxd.bolt, ~/.influxdbv2/engine, 

these 2 components can be configured using  -
influxd flags :  --bolt-path=~/.influxdbv2/influxd.bolt   ;   --engine-path=~/.influxdbv2/engine
Environment variable : INFLUXD_BOLT_PATH=~/.influxdbv2/influxd.bolt ; INFLUXD_ENGINE_PATH=~/.influxdbv2/engine
Configuration file : bolt-path: /users/user/.influxdbv2/influxd.bolt  ;  engine-path: /users/user/.influxdbv2/engine

Notes for v2:
writes handled via /api/v2/write endpoint or the /write 1.x compatibility endpoint.
reads handled via                         or the /read 
Data Point Batch --compressed--> WAL
                 --------------> Memory Cache ----periodically---> TSM file1 (disk)     ------
                                              -------------------> TSM file n           -----| --> hlTSM file (TSI?).




