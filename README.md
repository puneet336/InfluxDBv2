# Install and configure VictoriaMetrics on CentOS 7

## Setup influxdb service account
For running influxdb as non-root user, we need to create a victoriametrics user and group. To create a new system user and group, you can use these two commands:
```
groupadd -r myinfluxdb
useradd -g myinfluxdb -d /var/lib/myinfluxdb -s /sbin/nologin --system myinfluxdb
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
mkdir -p /usr/local/bin/influx261/
cp influxdb2_linux_amd64/influxd /usr/local/bin/influx261/
chown root:root /usr/local/bin/influx261/influxd
```

## Configure Influxdb

Create directories for VictoriaMetrics metadata, data , wal and set ownership to myinfluxdb account as :
```
mkdir -p /var/lib/myinfluxdb/meta
mkdir -p /var/lib/myinfluxdb/data
mkdir -p /var/lib/myinfluxdb/wal
chown -R myinfluxdb:myinfluxdb /var/lib/myinfluxdb/
```

setup logging directory 
```
mkdir -p /var/log/myinfluxdb/
chown myinfluxdb:myinfluxdb  /var/log/myinfluxdb/
```

Create new systemd services.
```
sudo vi /etc/systemd/system/myinfluxdb.service
```

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
Environment="INFLUXD_CONFIG_PATH=/usr/local/bin/influx261/influxd.conf"

ExecStart=/usr/local/bin/influx261/influxd run

ExecStop=/bin/kill -s SIGTERM $MAINPID

LimitNOFILE=65536
LimitNPROC=32000

[Install]
WantedBy=multi-user.target
```

create a configuration file /usr/local/bin/influx261/influxd.conf as - 
```
[http]
  bind-address = "0.0.0.0:9099"
  log-enabled = true
  write-tracing = false
  pprof-enabled = true
  auth-enabled = true
  shared-secret = "mysecret"
  max-row-limit = 0
  max-connection-limit = 0

[logging]
  format = "auto"
  level = "info"
  suppress-logo = false
  stderr = true
  log-file-path = "/var/log/myinfluxdb/influxdb2.log"
  log-rotate-enabled = true
  log-rotate-max-size = 10MB
  log-rotate-max-age = 7d

[meta]
  dir = "/var/lib/myinfluxdb/meta"

[storage]
  dir = "/var/lib/myinfluxdb/data"
  retention-intervals = ["1h", "6h", "12h", "24h", "2d", "7d", "30d", "365d"]
  retention-check-interval = "10m"
  wal-dir = "/var/lib/myinfluxdb/wal"

[[retry]]
  interval = "1s"
  max-interval = "10s"
  max-duration = "2m"
  exponential-base = 2
```

Enable and start VictoriaMetrics service to run on system boot automatically.

```
$ sudo systemctl enable victoriametrics.service --now
```





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




