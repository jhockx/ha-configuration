# Data storage
Currently I have a form of medium to long term storage setup. My SD card is 64GB and I backup snapshots to Google Drive. Since Google Drive has a maximum of 15GB free storage, which is less than the 64GB, Google Drive is the bottleneck. To reduce my data footprint, I'm down sampling all my data to one hourly intervals using Influxdb. The retention policy for this data is infinite, but for the raw data in the SQLite database of Home Assistant and for the raw data that is copied into Influxdb, the retention policy is 3 days. This means my snapshots will presumably grow bigger quite fast for the first three days, but after that the growth rate will be much lower. 

To do a back of the envelope calculation of the growth rate after the first three days, we can look at the [documentation of InfluxDB](https://docs.influxdata.com/influxdb/v1.8/guides/hardware_sizing/): *Non-string values require approximately three bytes. String values require variable space, determined by string compression*.

Per sensor I have one timestamp and one value, so ~6 bytes per sensor per hour. I have 49 sensors, which means that - in theory - each hour my data will grow with 49 * 6 = 294 bytes. To sum up, I made an overview of the growth rate over all important units of time:

| Unit of time | Growth rate (per unit of time) |
|--------------|--------------------------------|
| Hour         | 294 B                          |
| Day          | 7.1 kB                       |
| 30 Days      | 211.7 kB                      |
| Year         | 2.6 MB                     |

With ~2.6 MB of data each year, it looks like Google Drive will be full when I'm no longer here. That's why I'm not going to worry about it for now, but of course in practice it is very much possible that my snapshots will become bigger still, due to something unforeseen in Home Assistant. I'll keep an eye on it and depending on the results, I might start raising the resolution of the data (in case I have a lot of space left) or I might have to look at other solutions for making backups (in case I have too little space).

The rest of the sections will list my configuration for each component.

## Home Assistant
Add this to `configuration.yaml` to remove all data in SQLite after three days:
```
# Auto purge all data older than three days, long-term storage is done in influxdb
recorder:
  purge_keep_days: 3
```

## InfluxDB
Install the [InfluxDB Add-on](https://github.com/hassio-addons/addon-influxdb) and set the configuration:
```
auth: true
reporting: true
ssl: false
certfile: fullchain.pem
keyfile: privkey.pem
envvars: []
```

Add the following to `configuration.yaml` in Home Assistant:
```
# Long term storage and a few plots in Grafana 
influxdb:
  host: a0d7b954-influxdb # Go to the InfluxDB webpage and check the addressbar, replace _ with -
  port: 8086
  database: homeassistant
  username: homeassistant
  password: YOUR_PASSWORD
  max_retries: 3
  default_measurement: state
```

Create the user `homeassistant` in InfluxDB with password `YOUR_PASSWORD`. Now create the database by executing the following query:
```
CREATE DATABASE "homeassistant" WITH DURATION 3d SHARD DURATION 1d NAME "autogen"
```

Add the following Continuous Queries for long term storage:
```
CREATE CONTINUOUS QUERY "1h_kWh_cumulative" ON "homeassistant" 
BEGIN
  SELECT last("value") AS value
  INTO "infinite"."kWh"
  FROM "autogen"."kWh"
  WHERE "entity_id" =~ /daily|monthly|yearly/
  GROUP BY time(1h), "entity_id"
END

CREATE CONTINUOUS QUERY "1h_kWh" ON "homeassistant" 
BEGIN
  SELECT mean("value") AS value
  INTO "infinite"."kWh"
  FROM "autogen"."kWh"
  WHERE "entity_id" !~ /daily|monthly|yearly/
  GROUP BY time(1h), "entity_id"
END

CREATE CONTINUOUS QUERY "1h_m3" ON "homeassistant" 
BEGIN
  SELECT mean("value") AS value
  INTO "infinite"."m3"
  FROM "autogen"."m3"
  GROUP BY time(1h), "entity_id"
END

CREATE CONTINUOUS QUERY "1h_W" ON "homeassistant" 
BEGIN
  SELECT mean("value") AS value
  INTO "infinite"."W"
  FROM "autogen"."W"
  GROUP BY time(1h), "entity_id"
END

CREATE CONTINUOUS QUERY "1h_°C" ON "homeassistant" 
BEGIN
  SELECT mean("value") AS value
  INTO "infinite"."°C"
  FROM "autogen"."°C"
  GROUP BY time(1h), "entity_id"
END
```

## Google Drive
The `homeassistant` database in InfluxDB will be included in the snapshots of Home Assistant in its entirety. To backup this data, I have setup a backup to Google Drive for my snapshots. I'm using the [Hass.io Google Drive Backup Add-on](https://github.com/sabeechen/hassio-google-drive-backup) for this.
