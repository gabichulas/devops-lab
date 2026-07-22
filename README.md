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
                        (1. Upload Raw .mp3)     │               │ (2. Push Job Payload)
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



> [!note]
> The use of Kubernetes may be a clear overengineering, but the purpose of this project is just practicing and trying to build a production-like environment on the cloud.


---

### Subnet Layout
* **Public Subnets (`10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24`):** Hosts AWS NLB endpoints and NAT Gateways.
* **Private Subnets (`10.0.10.0/24`, `10.0.20.0/24`, `10.0.30.0/24`):** Hosts 3x K3s Master nodes and N x Worker nodes.
* **Control Plane HA:** 3-node embedded `etcd` quorum distributed across `us-east-1a`, `us-east-1b`, and `us-east-1c`.
* **Zero-Trust Administrative Boundary:** Inbound SSH is globally blocked. Node orchestration and Ansible playbooks execute strictly via **AWS Systems Manager (SSM) Session Manager**.

---

## Technical Stack

| Boundary | Technology | Purpose & Architectural Rationale |
| :--- | :--- | :--- |
| **IaC** | Terraform | Declarative multi-AZ VPC, S3 storage buckets, SQS queues, IAM roles, and security boundaries. |
| **Configuration Management** | Ansible | Idempotent system orchestration, K3s cluster coupling, and SSM-based transport execution. |
| **Container Orchestration** | K3s | Multi-AZ control plane running embedded `etcd` HA quorum. Minimal resource overhead. |
| **Asynchronous Message Queue** | AWS SQS | Decouples heavy HTTP uploads from TensorFlow model execution. Eliminates API timeout bottlenecks. |
| **Object Storage** | AWS S3 | Encrypted object store for raw `.mp3` uploads and 4-stem output artifacts (`vocals`, `drums`, `bass`, `other`). |
| **Event-Driven Autoscaling** | KEDA | Dynamically scales TensorFlow inference worker pods from `0` to `N` based on SQS queue depth. |
| **API Gateway** | FastAPI / Uvicorn | Asynchronous web interface for job submission, payload validation, and status tracking. |
| **ML Inference Engine** | TensorFlow / Keras | Containerized execution environment executing U-Net and Vision Transformer spectrogram transforms. |
| **Edge Ingress** | Traefik + AWS NLB | Pass-through Layer-4 load balancing at the cloud boundary mapped to in-cluster Layer-7 path routing. |

---

## Execution Workflow

1. The client uploads a `.mp3` file to the FastAPI Gateway specifying the target model architecture.
2. The Gateway generates a UUID `job_id`, uploads the raw audio to the input S3 bucket, and pushes a structured JSON message to AWS SQS. The HTTP response immediately returns `202 Accepted`.
3. KEDA detects SQS queue depth > 0 via AWS CloudWatch/SQS APIs and triggers worker deployment scaling from `0` to `N`.
4. **TensorFlow Inference Execution:**
   * A worker pod claims the message (initiating SQS visibility timeout).
   * Downloads the raw `.mp3` track from S3.
   * Performs STFT audio preprocessing.
   * Passes the spectrogram through the loaded TensorFlow model weights.
   * Reconstructs the 4 output audio stems using Inverse STFT.
5. The worker uploads `vocals.mp3`, `drums.mp3`, `bass.mp3`, and `other.mp3` to the S3 destination path `stems/<job_id>/`, acknowledges the SQS message, and terminates or awaits the next job.

---

## License

Distributed under the MIT License. See `LICENSE` for more information.
