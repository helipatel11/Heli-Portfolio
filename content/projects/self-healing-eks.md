---
title: "Self-Healing EKS Cluster"
date: 2025-11-17
description: "Building a Self-Healing EKS Cluster with Terraform and Cluster Autoscaler"
---


## Introduction

Running production-grade Kubernetes clusters requires **resiliency, scalability, and automation**. In this project, I built a **self-healing Amazon EKS cluster** using **Terraform** and **Cluster Autoscaler**, demonstrating how infrastructure can automatically recover and scale based on workload demands.

The project highlights:

* Automated provisioning of **VPC, EKS cluster, and node groups** using Terraform
* Dynamic scaling of worker nodes with **Cluster Autoscaler**
* Resilient workloads that **self-heal** when nodes fail or pods are evicted
* Spot and on-demand instance management for cost optimization

This project showcases a **production-ready cloud infrastructure workflow** suitable for portfolios targeting Cloud, Kubernetes, or DevOps roles.

---

## Architecture Overview

![Architecture](/images/Self-Healing-EKS-Cluster.png)



**Diagram Explanation:**

* **Terraform** provisions VPC, subnets, and the EKS cluster with managed node groups.
* **Cluster Autoscaler** observes node utilization and scales the cluster dynamically.
* **Workloads** (e.g., `php-apache`) automatically recover on node failure.
* **IAM Policies** give the autoscaler necessary permissions to manage AWS resources.

---

## Implementation Highlights

### 1. Terraform Infrastructure

* **VPC & Subnets:** Public, private, and intra subnets for cluster networking
* **EKS Cluster & Managed Node Groups:** Supports **spot and on-demand instances**
* **IAM Policies:** Cluster Autoscaler has secure permissions to adjust ASGs

---

### 2. Cluster Autoscaler

* Deployed in the `kube-system` namespace with proper **ServiceAccount and RBAC**
* Scales nodes **up or down based on pending pods** and utilization metrics
* Uses **least-waste expander strategy** to optimize cost

---

### 3. Workload Testing

* Deploy a **sample PHP-Apache app** with multiple replicas to simulate load
* Autoscaler adjusts node count dynamically based on resource requests
* Nodes can be terminated manually or via spot interruption to test **self-healing**

---

### 4. Self-Healing Behavior

* Kubernetes automatically reschedules pods when nodes fail
* Cluster maintains **desired capacity** and recovers workloads
* Spot instance interruptions are handled gracefully (can add AWS Node Termination Handler for production)

---

## Key Takeaways

* Combining **Terraform, EKS, and Cluster Autoscaler** creates resilient, scalable infrastructure
* Node tagging and IAM policies ensure **secure and automated scaling**
* Testing workloads validates **autoscaling and self-healing** capabilities
* Spot instance usage optimizes cost without sacrificing reliability

---

## Future Improvements

* **AWS Node Termination Handler:** Handles spot instance interruptions gracefully
* **Observability Stack (Prometheus/Grafana):** Monitor autoscaler behavior, logs, and metrics
* **CI/CD for Terraform:** Automate infrastructure updates with GitHub Actions or GitOps
* **PodSecurity & Network Policies:** Improve cluster security and compliance
* **Multi-AZ / Multi-Region:** For higher availability and fault tolerance

---

## How to Run / Test

```bash
# Initialize Terraform
terraform init

# Review the plan
terraform plan

# Apply infrastructure
terraform apply

# Deploy a sample workload
kubectl apply -f finaldep.yaml

# Check Cluster Autoscaler logs
kubectl -n kube-system logs deploy/cluster-autoscaler
```
