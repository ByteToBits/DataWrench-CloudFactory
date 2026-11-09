# DataWrench: Cloud Factory

![Go](https://img.shields.io/badge/Go-00ADD8?style=flat&logo=go&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Apache Kafka](https://img.shields.io/badge/Apache_Kafka-231F20?style=flat&logo=apache-kafka&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat&logo=redis&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat&logo=postgresql&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat&logo=ansible&logoColor=white)
![Red Hat](https://img.shields.io/badge/RHEL-EE0000?style=flat&logo=redhat&logoColor=white)

This repository contains my open-source passion project for a Manufacturing Execution System (MES). It includes all backend services for the Kubernetes clusters, along with other resources such as the SQL schema and SEC/GEM host. The project provides a scalable and flexible MES solution, ranging from small-scale manufacturing to critical high-fault-tolerant, high-throughput semiconductor manufacturing.

Front-End Demo:

YouTube Demo:

Documentation & Resources: [Google Drive](https://drive.google.com/drive/folders/1IlRub_8c6jkS4ydqFX7AeLRfFl0_Vmcx?usp=sharing)

## System Architecture
The project adopts a Microservice Architecture to allow for scalable resources and services depending on project requirements and budget. It can be deployed on-premises or on the cloud, and scaled in performance by adding more nodes. Fault tolerance is achieved by adopting multiple redundancy strategies to ensure critical fab production uptime.

![System Architecture](https://drive.google.com/uc?export=view&id=169hLpwP73VGz7dwqfqrOWhszBZSwi2_U)

## Homelab Architecture
The project is designed in such a manner that it can be deployed on-premises or on the cloud, depending on project requirements. It adopts a Secure-by-Design principle by applying industry standard designs to ensure compliance with IEC 62443 and ISA-95 Purdue Model.

However, for the actual deployment, the project will use a Homelab Architecture to facilitate the development of the project. This architecture will also resemble the on-premises deployment in actual production time.

![Homelab Architecture](https://drive.google.com/uc?export=view&id=1vyy9Ax2i-IHAYhs1DvDP_zFQer5omg3S)

## Kubernetes Architecture
The MES microservices run on a vanilla Kubernetes cluster built with kubeadm, with one control plane and one worker node on each hypervisor so the cluster survives the loss of a whole host. HAProxy and a keepalived virtual IP give every node a single, highly available address for the Kubernetes API, whilst MetalLB provides the application virtual IP and Envoy Gateway routes each request to the right microservice. Calico provides the pod network and network policies, and the cluster connects to the database, cache and message broker running on their own virtual machines.

![Kubernetes Architecture](https://drive.google.com/uc?export=view&id=1oK0W2USHlmCc0CeYx1scuRzu6Mb3o_2h)

## Database Architecture
The database runs on a three-node PostgreSQL cluster with TimescaleDB to handle large volumes of time-series data, with one node on each hypervisor. Patroni and etcd handle leader election and automatic failover, whilst HAProxy and a keepalived virtual IP give applications a single address that always reaches the primary database. 

![Database Architecture](https://drive.google.com/uc?export=view&id=1QfIdeIOAGL6L_Q9pPJc1wMV5YD75xSr8)