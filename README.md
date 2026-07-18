# DataWrench: Cloud Factory

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Apache Kafka](https://img.shields.io/badge/Apache_Kafka-231F20?style=flat&logo=apache-kafka&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat&logo=redis&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)

This repository contains my open-source passion project for a Manufacturing Execution System (MES). It includes all backend services for the Kubernetes clusters, along with other resources such as the SQL schema and SEC/GEM host. The project provides a scalable and flexible MES solution, ranging from small-scale manufacturing to critical high-fault-tolerant, high-throughput semiconductor manufacturing.

Front-End Demo: 

YouTube Demo:

## System Architecture
The project adopts a Microservice Architecture to allow for scalable resources and services depending on project requirements and budget. It can be deployed on-premises or on the cloud, and scaled in performance by adding more nodes. Fault tolerance is achieved by adopting multiple redundancy strategies to ensure critical fab production uptime.

![System Architecture](CloudFactory/doc/System%20Architecture.jpg)

## Homelab Architecture
For the actual deployment, the project will use a Homelab Architecture to facilitate the development of the project. This architecture will also resemble the on-premises deployment in actual production time.

![Homelab Architecture](https://drive.google.com/file/d/1yyqAfiGvXpWQoqro-moSym82BkwaIWzq/view?usp=sharing)

## Commerical Network Architecture
In addition, the project is designed in such a manner that it can be deployed on-premises or on the cloud, depending on project requirements. It adopts a Secure-by-Design principle by applying industry standard designs to ensure compliance with IEC 62443 and ISA-95 (Purdue Model).

![Network Architecture](CloudFactory/doc/Network%20Architecture.jpg)

