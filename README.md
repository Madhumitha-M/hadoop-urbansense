# UrbanSense – Hadoop Demo (Group 2)

Smart city IoT analytics platform on Apache Hadoop.  

---

## Prerequisites

- Docker Desktop installed and running
- At least 6 GB of free RAM
- Ports free: 9870, 9000, 9864, 9865, 8088, 8188

---

## Start the Cluster

```bash
# Start all containers
docker compose up -d

# Wait 60–90 seconds, then verify
docker compose ps
```

All 5 containers should show status: Up

---

## NameNode Web UI

Open browser → http://localhost:9870  
Check: Live Nodes = 2, Dead Nodes = 0


## Create HDFS Directory Structure

```bash
#late data
docker exec namenode hdfs dfs -mkdir -p /urbansense/raw/air_quality/Ost/2026/04/17/22 
#live data
docker exec namenode hdfs dfs -mkdir -p /urbansense/raw/air_quality/Ost/2026/04/21/22 
#merge data
docker exec namenode hdfs dfs -mkdir -p /urbansense/public/serving/air_quality/Ost 
#upload 200mb file
docker exec namenode hdfs dfs -mkdir -p /urbansense/raw/air_quality/bulk/ 
#Op2
docker exec namenode hdfs dfs -mkdir -p /urbansense/ingestion/raw/Ost/ 

# Verify all folders created
docker exec namenode hdfs dfs -ls /urbansense/
```
```bash
#print entire structure
docker exec namenode hdfs dfs -ls -R /urbansense/ 
```
---

## Live Demo – 3 Operations

### Operation 1 – Late Arrival (Veracity)
**What it demonstrates:**  
A sensor was offline for 72 hours. When it reconnects, late data is routed  
to the correct historical partition. 
Lambda Architecture then processes both old and new data simultaneously.
```bash
#old late data
docker exec namenode hdfs dfs -put /demo-data/sensor_late.csv /urbansense/raw/air_quality/Ost/2026/04/17/22/  
docker exec namenode hdfs dfs -ls /urbansense/raw/air_quality/Ost/2026/04/17/22/
#New live data
docker exec namenode hdfs dfs -put /demo-data/sensor_live.csv /urbansense/raw/air_quality/Ost/2026/04/21/22/  
docker exec namenode hdfs dfs -ls /urbansense/raw/air_quality/Ost/2026/04/21/22/
## Run .py 
#merge both
python serving_layer.py 
#upload merged data
docker exec namenode hdfs dfs -put /demo-data/final_serving_view.csv /urbansense/public/serving/air_quality/Ost/ 
#final output
docker exec namenode hdfs dfs -cat /urbansense/public/serving/air_quality/Ost/final_serving_view.csv 
```

### Operation 2 - Schema-On-Read - Hadoop's core principle
**what it demonstrates**
Store everything raw, in its original format. Apply the schema only when you read/process it downstream.
 
```bash
#json over MQTT
docker exec namenode hdfs dfs -put /demo-data/sensor_mqtt.json /urbansense/ingestion/raw/Ost/ 
#csv from SFTP 
docker exec namenode hdfs dfs -put /demo-data/sensor_sftp.csv /urbansense/ingestion/raw/Ost/ 
#txt from GTFS-RT
docker exec namenode hdfs dfs -put /demo-data/sensor_gtfs.txt /urbansense/ingestion/raw/Ost/ 
#show all 3
docker exec namenode hdfs dfs -ls /urbansense/ingestion/raw/Ost/ 
#verify readable
docker exec namenode hdfs dfs -cat /urbansense/ingestion/raw/Ost/sensor_mqtt.json 
docker exec namenode hdfs dfs -cat /urbansense/ingestion/raw/Ost/sensor_gtfs.txt
```
### Operation 3 - Block distribution and NameNode Web UI
**What it demonstrates:**  
How hdfs splits a large file into 128MB blocks and distributes replicas across multiple DataNodes.
```bash
#generate file
docker exec namenode dd if=/dev/urandom of=/demo-data/sensor_large.bin bs=1M count=200
#upload file
docker exec namenode hdfs dfs -put /demo-data/sensor_large.bin /urbansense/raw/air_quality/bulk/
#verify
docker exec namenode hdfs dfs -ls /urbansense/raw/air_quality/bulk/ 
#block distribution(upload takes some time)
docker exec namenode hdfs fsck /urbansense/raw/air_quality/bulk/sensor_large.bin -files -blocks -locations 
#### NameNode Web UI - http://localhost:9870 → Utilities → Browse /urbansense/raw/air_quality/bulk/ 
```
---

## Stop the Cluster

```bash
docker compose down # Stop but keep data
docker compose down -v # full reset
```