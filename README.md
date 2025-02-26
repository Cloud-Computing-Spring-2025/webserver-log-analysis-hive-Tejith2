# Web Server Log Analysis with Apache Hive

## **Project Overview**
This project implements a **Hive-based analytical process** for analyzing web server logs. The goal is to process a dataset of **web server logs (CSV format)** and extract meaningful insights about website traffic patterns. The analysis includes:

- **Total Web Requests**
- **HTTP Status Code Analysis**
- **Most Visited Pages**
- **Traffic Source Analysis (User Agents)**
- **Suspicious IP Detection**
- **Traffic Trend Over Time**
- **Optimizing Query Performance using Partitioning**
- **Exporting Results for Further Processing**

The **output is stored in HDFS** and transferred to the local machine for final reporting.

---

## **Implementation Approach**
### **Hive Table Creation**

A Hive **external table** was created to store web server logs. **Backticks (`) were used to escape the `timestamp` column, as it is a reserved keyword.**

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS web_server_logs (
    ip STRING,
    `timestamp` STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
TBLPROPERTIES ("skip.header.line.count"="1");
```

### **Data Loading from HDFS**
Data was first uploaded from the **local system** to **HDFS**:

```bash
docker cp web_server_logs.csv namenode:/web_server_logs.csv
docker exec -it namenode /bin/bash
hdfs dfs -mkdir -p /data
hdfs dfs -put /web_server_logs.csv /data/
```

Then, the data was loaded into **Hive**:

```sql
LOAD DATA INPATH '/data/web_server_logs.csv' INTO TABLE web_server_logs;
```

### **Querying Web Server Logs**
#### 1️⃣ **Total Web Requests**
```sql
SELECT COUNT(*) AS total_requests FROM web_server_logs;
```

#### 2️⃣ **Status Code Analysis**
```sql
SELECT status, COUNT(*) AS count
FROM web_server_logs
GROUP BY status
ORDER BY count DESC;
```

#### 3️⃣ **Most Visited Pages**
```sql
CREATE TABLE most_visited_temp AS
SELECT url, COUNT(*) AS visit_count
FROM web_server_logs
GROUP BY url;

INSERT OVERWRITE DIRECTORY '/data/output_most_visited'
SELECT url, visit_count
FROM most_visited_temp
ORDER BY visit_count DESC
LIMIT 3;
```

#### 4️⃣ **Traffic Source Analysis (User Agents)**
```sql
CREATE TABLE traffic_sources_temp AS
SELECT user_agent, COUNT(*) AS request_count
FROM web_server_logs
GROUP BY user_agent;

INSERT OVERWRITE DIRECTORY '/data/output_traffic_sources'
SELECT user_agent, request_count
FROM traffic_sources_temp
ORDER BY request_count DESC
LIMIT 3;
```

#### 5️⃣ **Suspicious IP Addresses (More Than 3 Failed Requests)**
```sql
SELECT ip, COUNT(*) AS failed_requests
FROM web_server_logs
WHERE status IN (404, 500)
GROUP BY ip
HAVING COUNT(*) > 3;
```

#### 6️⃣ **Traffic Trend Over Time**
```sql
SELECT SUBSTR(`timestamp`, 1, 16) AS time, COUNT(*) AS request_count
FROM web_server_logs
GROUP BY SUBSTR(`timestamp`, 1, 16)
ORDER BY time;
```

#### 7️⃣ **Optimizing Query Performance Using Partitioning**
```sql
CREATE TABLE IF NOT EXISTS web_server_logs_partitioned (
    ip STRING,
    `timestamp` STRING,
    url STRING,
    user_agent STRING
)
PARTITIONED BY (status INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;

INSERT INTO TABLE web_server_logs_partitioned PARTITION(status)
SELECT ip, `timestamp`, url, user_agent, status FROM web_server_logs;
```

---

## **Execution Steps**
### **1️⃣ Start Hive Environment**
```bash
docker-compose up -d
docker exec -it hive-server /bin/bash
hive
```

### **2️⃣ Upload Data to HDFS**
```bash
docker cp web_server_logs.csv namenode:/web_server_logs.csv
docker exec -it namenode /bin/bash
hdfs dfs -mkdir -p /data
hdfs dfs -put /web_server_logs.csv /data/
```

### **3️⃣ Run Hive Queries**
- Run each **analysis query** inside Hive CLI.

### **4️⃣ Retrieve Results from HDFS**
```bash
hdfs dfs -get /data/output_* /root/
```

### **5️⃣ Copy Results from Namenode to Local Machine**
```bash
docker cp namenode:/root/output_total_requests.txt ./
docker cp namenode:/root/output_status_codes.txt ./
docker cp namenode:/root/output_most_visited.txt ./
docker cp namenode:/root/output_traffic_sources.txt ./
docker cp namenode:/root/output_suspicious_ips.txt ./
docker cp namenode:/root/output_traffic_trends.txt ./
```

---

## **Challenges Faced & Solutions**
### **1. Hive Partitioning Error**
**Issue:** Dynamic partitioning was not enabled.
**Fix:** Set Hive properties:
```sql
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;
```

### **2. Aggregate Function in INSERT OVERWRITE DIRECTORY Error**
**Issue:** Hive **does not support UDAF (COUNT) inside INSERT OVERWRITE DIRECTORY**.
**Fix:** Create a temporary table and insert sorted results.

### **3. Copying Files from HDFS to Local Machine**
**Issue:** Hive stores results in `000000_0` inside directories.
**Fix:** Move files and rename them before copying:
```bash
mv /root/output_most_visited/000000_0 /root/output_most_visited.txt
```

---

## **Sample Input & Expected Output**
### **Input Sample (web_server_logs.csv)**
```
ip,timestamp,url,status,user_agent
192.168.1.1,2024-02-01 10:15:00,/home,200,Mozilla/5.0
192.168.1.2,2024-02-01 10:16:00,/products,200,Chrome/90.0
192.168.1.3,2024-02-01 10:17:00,/checkout,404,Safari/13.1
```

### **Expected Output Example**
```
Total Requests: 100

Status Code Analysis:
200: 80
404: 10
500: 10

Most Visited Pages:
/home: 50
/products: 30
/checkout: 20

Suspicious IPs:
192.168.1.10: 5 failed requests
192.168.1.15: 4 failed requests
```

---

## **Conclusion**
This project successfully implements **Apache Hive for log analysis**. It enables efficient **data querying, optimization via partitioning, and exporting structured insights**. The approach ensures **scalability and performance efficiency for handling large-scale log data**.

