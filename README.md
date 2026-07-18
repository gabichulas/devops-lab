# AWS Multi-Node Kubernetes Cluster

# Status: WIP

This repository contains the configurations and automation playbooks required to provision, configure, and operate a bare-metal equivalent Multi-Node Kubernetes cluster on AWS.

Instead of relying on managed services like Amazon EKS, this project implements a highly cost-effective and operationally transparent architecture utilizing Terraform for resource provisioning, Ansible for operating system orchestration, and [K3s](https://k3s.io/) as the lightweight Kubernetes distribution using its native, built-in Traefik Ingress Controller.

---

## Architecture Overview

The infrastructure model adheres to a standard control-plane and worker-node topology deployed within a dedicated, isolated Virtual Private Cloud (VPC).

                                                  [ External Traffic ]
                                                           │
                                                           ▼
                                                ┌─────────────────────┐
                                                │  Traefik Ingress    │
                                                │   (Native K3s)      │
                                                └──────────┬──────────┘
                                                           │ (Layer 7 Routing)
                                           ┌───────────────┴───────────────┐
                                           ▼                               ▼
                                ┌─────────────────────┐         ┌─────────────────────┐
                                │    alpha-service    │         │    beta-service     │
                                │     (ClusterIP)     │         │     (ClusterIP)     │
                                └──────────┬──────────┘         └──────────┬──────────┘
                                           │                               │
                                   ┌───────┴───────┐               ┌───────┴───────┐
                                   ▼               ▼               ▼               ▼
                          ┌───────────────┐┌───────────────┐┌───────────────┐┌───────────────┐
                          │ k3s-worker-1  ││ k3s-worker-2  ││ k3s-worker-1  ││ k3s-worker-2  │
                          │ ┌───────────┐ ││ ┌───────────┐ ││ ┌───────────┐ ││ ┌───────────┐ │
                          │ │ alpha-pod │ ││ │ alpha-pod │ ││ │ beta-pod  │ ││ │ beta-pod  │ │
                          │ └───────────┘ ││ └───────────┘ ││ └───────────┘ ││ └───────────┘ │
                          └───────────────┘└───────────────┘└───────────────┘└───────────────┘

* Networking: 1x Dedicated VPC, 1x Public Subnet, 1x Internet Gateway, and an optimized Security Group ensuring tight internal cluster communication (10.0.0.0/16) while exposing strictly necessary boundaries (SSH, HTTP/S, and TLS for the API Server).
* Control Plane: 1x Ubuntu 22.04 LTS instance hosting the K3s API server, controller manager, scheduler, and an embedded SQLite datastore.
* Data Plane: 2x Ubuntu 22.04 LTS instances running the K3s agent runtime.
* Ingress & Routing: Edge routing handled seamlessly by the pre-configured Traefik ingress service, executing Layer 7 path-based discrimination across decoupled internal target groups (/alpha and /beta).

---
