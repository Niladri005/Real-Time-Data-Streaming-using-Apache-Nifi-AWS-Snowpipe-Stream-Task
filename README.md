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

```
flowchart LR
    A[Python Faker\nFake Customer Data] --> B[Apache NiFi\n(ListFile → FetchFile → PutS3Object)]
    B --> C[S3 Bucket\n(real-time data landing)]
    C --> D[Snowflake External Stage]
    D --> E[Snowflake Tables\nRaw, Current, History]
    E --> F[Snowflake Tasks & Streams\nSCD1 & SCD2 handling]
    F --> G[JupyterLab / BI Tools\nAnalysis & Visualization]

```
