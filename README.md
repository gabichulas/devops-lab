<p align="center">
  <h1 align="center">Demix Cloud</h1>
  <p align="center">
    <strong>Distributed, Event-Driven Audio Source Separation Platform on Multi-AZ Kubernetes</strong>
  </p>
  <p align="center">
    <a href="https://github.com/gabichulas/demix-cloud/stargazers"><img src="https://img.shields.io/github/stars/gabichulas/demix-cloud?style=for-the-badge&color=gold" alt="Stars"></a>
    <a href="https://github.com/gabichulas/demix-cloud/blob/main/LICENSE"><img src="https://img.shields.io/badge/License-MIT" alt="License"></a>
    <img src="https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white" alt="AWS">
    <img src="https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white" alt="Terraform">
    <img src="https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white" alt="Ansible">
    <img src="https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" alt="Kubernetes">
    <img src="https://img.shields.io/badge/TensorFlow-FF6F00?style=for-the-badge&logo=tensorflow&logoColor=white" alt="TensorFlow">
    <img src="https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white" alt="FastAPI">
  </p>
</p>

---

## Summary

Demix Cloud is an event-driven MLOps platform designed to run asynchronous audio source separation workloads at scale. Built as an infrastructure extension of the **Demix** research project, the system evaluates and benchmarks **U-Net** vs. **Vision Transformer (ViT)** architectures implemented in **TensorFlow** for stem extraction (vocals, drums, bass, other).

The underlying compute platform bypasses managed Kubernetes services (EKS) in favor of a cost-optimized, highly available **[K3s](https://k3s.io/) cluster**. The entire infrastructure lifecycle—spanning VPC network isolation across 3 Availability Zones, storage buckets, message queues, OS-level hardening via AWS Systems Manager (SSM), and KEDA-driven worker pod autoscaling is fully automated using **Terraform** and **Ansible**.

> [!note]
> As said, this project is an extension of [Demix](https://github.com/gabichulas/demix), an university research project. For more details about it, visit the repo page.

---

## Architecture Topology

                                            [ External Audio Ingress ]
                                                         │
                                                         ▼
                                         ┌───────────────────────────────┐
                                         │            AWS NLB            │
                                         └───────────────┬───────────────┘
                                                         │ 
                                                         ▼
                                         ┌───────────────────────────────┐
                                         │ Traefik Ingress (Private Sub) │
                                         └───────────────┬───────────────┘
                                                         │
                                                         ▼
                                         ┌───────────────────────────────┐
                                         │     FastAPI Gateway Pods      │
                                         └───────┬───────────────┬───────┘
                                                 │               │
                        (1. Upload Raw .wav)     │               │ (2. Push Job Payload)
                                                 ▼               ▼
                                     ┌───────────────┐       ┌───────────────┐
                                     │  AWS S3       │       │  AWS SQS      │
                                     │  (Raw Audio)  │       │  (Jobs Queue) │
                                     └───────────────┘       └───────┬───────┘
                                                                     │
                                                                     │ (3. Queue Depth Metric)
                                                                     ▼
                                                             ┌───────────────┐
                                                             │  KEDA Scaler  │
                                                             └───────┬───────┘
                                                                     │
                                                                     │ (4. Auto-Scale 0 -> N)
                                                                     ▼
                                                     ┌───────────────────────────────┐
                                                     │   Demix TensorFlow Workers    │
                                                     │   (U-Net & ViT Inference)     │
                                                     └───────────────┬───────────────┘
                                                                     │
                                                                     │ (5. Persist Separated Stems)
                                                                     ▼
                                                             ┌───────────────┐
                                                             │  AWS S3       │
                                                             │ (Output Stems)│
                                                             └───────────────┘
