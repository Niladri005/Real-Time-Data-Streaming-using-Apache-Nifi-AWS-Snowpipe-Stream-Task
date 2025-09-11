# Real-Time Data Streaming with AWS EC2, Apache NiFi, S3, and Snowflake  

![AWS](https://img.shields.io/badge/AWS-EC2-orange?logo=amazon-aws&logoColor=white)  
![Docker](https://img.shields.io/badge/Docker-Containerization-blue?logo=docker&logoColor=white)  
![Apache NiFi](https://img.shields.io/badge/Apache-NiFi-green?logo=apache&logoColor=white)  
![Snowflake](https://img.shields.io/badge/Snowflake-Data%20Warehouse-29B5E8?logo=snowflake&logoColor=white)  
![Python](https://img.shields.io/badge/Python-Faker-yellow?logo=python&logoColor=white)  

---

## 📌 Project Overview  
This project demonstrates a **real-time data streaming pipeline** using:  

- **AWS EC2** for infrastructure  
- **Docker** for containerized services (Apache NiFi, JupyterLab, Zookeeper)  
- **Python Faker** for synthetic data generation  
- **Apache NiFi** for real-time ETL (Extract → Transform → Load)  
- **AWS S3** as staging storage  
- **Snowflake Data Warehouse** for downstream analytics & Slowly Changing Dimension (SCD) handling  

The goal: **Ingest real-time customer data → land into S3 → auto-ingest into Snowflake → maintain historical and current states (SCD1 & SCD2).**

---

## ⚙️ Project Architecture  

![Architecture Diagram](https://github.com/Niladri005/Real-Time-Data-Streaming-using-Apache-Nifi-AWS-Snowpipe-Stream-Task/blob/main/real-time%20streaming.jpg)

```
flowchart LR
    A[Python Faker\nFake Customer Data] --> B[Apache NiFi\n(ListFile → FetchFile → PutS3Object)]
    B --> C[S3 Bucket\n(real-time data landing)]
    C --> D[Snowflake External Stage]
    D --> E[Snowflake Tables\nRaw, Current, History]
    E --> F[Snowflake Tasks & Streams\nSCD1 & SCD2 handling]
    F --> G[JupyterLab / BI Tools\nAnalysis & Visualization]

```

## 🚀 Step 1: AWS EC2 Setup

- ** Instance type: t2.xlarge

- ** Storage: 32GB (recommended)

- ** Security Group: Allow ports 4000-38888 (for NiFi, Jupyter, etc.)

### Connect to EC2
```
ssh -i snowflake-project.pem ec2-user@<ec2-public-dns>
```
### Copy project files to EC2
```
scp -r -i snowflake-project.pem docker-exp ec2-user@<ec2-public-dns>:/home/ec2-user/docker_exp
```

## 🐳 Step 2: Install Docker & Docker Compose:
```
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker

# Install docker-compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Add user to docker group
sudo gpasswd -a $USER docker
newgrp docker

# Install pip + docker-compose (alt method)
sudo yum install python-pip -y
sudo pip install docker-compose
```

## 🧩 Step 3: Docker Compose Setup
```
version: "3.6"
volumes:
  shared-workspace:
    name: "hadoop-distributed-file-system"
    driver: local

services:
  jupyterlab:
    image: pavansrivathsa/jupyterlab
    hostname: JUPYTERLAB
    container_name: jupyterlab
    ports:
      - 4888:4888
      - 4040:4040
      - 8050:8050
    volumes:
      - shared-workspace:/opt/workspace

  zookeeper:
    image: bitnami/zookeeper
    hostname: ZOOKEEPER
    container_name: zookeeper
    ports:
      - 2181:2181
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes

  nifi:
    image: apache/nifi:1.14.0
    hostname: NiFi
    container_name: nifi
    ports:
      - 2080:2080
    environment:
      - NIFI_WEB_HTTP_PORT=2080
      - NIFI_CLUSTER_IS_NODE=true
      - NIFI_CLUSTER_NODE_PROTOCOL_PORT=2084
      - NIFI_ZK_CONNECT_STRING=zookeeper:2181
      - NIFI_ELECTION_MAX_WAIT=1 min
      - NIFI_SENSITIVE_PROPS_KEY=vvvvvvvvvvvv
    volumes:
      - shared-workspace:/opt/workspace/nifi
```

### Access Services

- ** NiFi: http://<EC2-IP>:2080/nifi/

- ** Jupyter Lab: http://<EC2-IP>:4888/lab


## 📝 Step 4: Generate Fake Real-Time Data
```
from faker import Faker
import csv, random
from datetime import datetime

RECORD_COUNT = 10000
fake = Faker()

with open(f"customer_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv", "w") as f:
    writer = csv.writer(f)
    writer.writerow(["customer_id","first_name","last_name","email","street","city","state","country"])
    for i in range(RECORD_COUNT):
        writer.writerow([
            i+1, fake.first_name(), fake.last_name(),
            fake.email(), fake.street_address(),
            fake.city(), fake.state(), fake.country()
        ])
```


## 🔄 Step 5: Apache NiFi Flow

### Three processors configured:

-ListFile – Detects new fake CSV files

-FetchFile – Reads file contents

-PutS3Object – Uploads to S3 bucket
![Architecture Diagram](https://github.com/Niladri005/Real-Time-Data-Streaming-using-Apache-Nifi-AWS-Snowpipe-Stream-Task/blob/main/nifi_flow_processor.jpg)

## 🗄️ Step 6: Snowflake Integration
```
create database if not exists scd_demo;
create schema if not exists scd2;

create or replace table customer (...);
create or replace table customer_history (...);
create or replace table customer_raw (...);

create or replace stream customer_table_changes on table customer;
```

### External Stage & Pipe

```
CREATE OR REPLACE STAGE customer_ext_stage
  url='s3://real-time-nifi-data/stream-data/'
  credentials=(aws_key_id='xxx' aws_secret_key='xxx');

CREATE OR REPLACE FILE FORMAT CSV TYPE = CSV FIELD_DELIMITER = "," SKIP_HEADER = 1;

CREATE OR REPLACE PIPE customer_s3_pipe AUTO_INGEST = TRUE AS
COPY INTO customer_raw FROM @customer_ext_stage FILE_FORMAT = CSV;

```

## 🔄 Step 7: Slowly Changing Dimensions
### SCD1 (Overwrite strategy)

- Merge customer_raw → customer

- Update changed fields with latest values

- Automate with Snowflake Procedure + Task

### SCD2 (History tracking)

- Use customer_table_changes stream

- Maintain customer_history with is_current, start_time, end_time

- Automated with scheduled Snowflake task

## ✅ Key Learnings

- Dockerized setup makes NiFi, Jupyter, and Zookeeper portable & reproducible.

- NiFi simplifies real-time ingestion pipelines.

- Snowflake Streams + Tasks provide powerful Change Data Capture (CDC) & SCD handling.

- End-to-end pipeline ensures real-time ingestion → historical data warehousing → analytics readiness.
