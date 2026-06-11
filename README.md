# Real-World DevOps / SRE Incidents

This repository documents production-style DevOps, SRE, DevSecOps, networking, observability, and cloud deployment incidents reproduced in my homelab.

The goal is to demonstrate practical ability to detect, troubleshoot, resolve, and prevent real-world operational failures.

## Homelab Focus Areas

- Kubernetes failure recovery
- GitLab CI/CD deployment issues
- Private container registry troubleshooting
- Terraform drift and state management
- Prometheus/Grafana observability
- Wazuh, Falco, Kyverno, and DevSecOps controls
- DNS, firewall, VLAN, and VPN troubleshooting
- Cloud deployment on Azure and AWS

## Incident Catalog

| ID | Incident | Domain | Status |
|---|---|---|---|
| 001 | ImagePullBackOff due to Private GitLab Registry Authentication Failure | Kubernetes / Registry / DevSecOps | Completed |
| 002 | CrashLoopBackOff due to Missing Application Configuration | Kubernetes | Planned |
| 003 | Ingress 502 due to Wrong Service Target Port | Kubernetes / Networking | Planned |
| 004 | Terraform Drift due to Manual Cloud Resource Change | Terraform | Planned |
| 005 | Prometheus High Cardinality Metrics | Observability | Planned |

## Completed Incident

### Incident 001: ImagePullBackOff due to Private GitLab Registry Authentication Failure

This incident reproduces a Kubernetes workload failure where a pod cannot pull an image from a private GitLab Container Registry.

Key learning points:

- Kyverno blocked the initial deployment because the image used the `latest` tag.
- The manifest was corrected to use an explicit image tag.
- Kubernetes then admitted the workload.
- The pod failed with `ImagePullBackOff` because it attempted anonymous access to a private GitLab registry.
- The issue was fixed by creating a Kubernetes Docker registry secret using a GitLab Deploy Token.
- The Deployment was patched with `imagePullSecrets`.
- The pod successfully recovered and reached `Running` state.

## Incident Documentation Format

Each incident includes:

- Failure simulation
- Detection method
- Troubleshooting commands
- Root cause analysis
- Resolution steps
- Validation
- Preventive actions
- Evidence
- Lessons learned

## Evidence Policy

Sensitive values are not committed.

This repository does not include:

- GitLab deploy tokens
- Passwords
- Kubernetes Secret YAML containing encoded credentials
- Private keys
- Sensitive screenshots
