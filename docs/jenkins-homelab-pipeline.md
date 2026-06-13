# Jenkins Homelab Pipeline Validation

This repository includes a Jenkins Pipeline from SCM to validate CI execution on a dedicated homelab Jenkins inbound agent.

## Architecture

GitHub Repository
→ Jenkins Controller on Mac
→ Jenkins Inbound Agent VM
→ Pipeline Execution

## Jenkins Agent

- Agent type: Inbound agent
- Agent hostname: jenkinsagent
- Runtime user: jenkins
- Agent service: systemd
- Jenkins label: homelab
- Controller URL: http://192.168.0.46:8080

## Pipeline Stages

1. Checkout SCM
2. Verify Homelab Agent
3. Validate Repository Structure

## Validation Coverage

The pipeline validates that:

- Jenkins can pull the repository from GitHub
- Jenkins can read the Jenkinsfile from SCM
- The job runs on the homelab Jenkins agent
- The repository contains required documentation
- The repository structure is visible from the Jenkins workspace

## Operational Notes

The Jenkins agent runs as a persistent systemd service on the homelab VM.

Useful commands:

- sudo systemctl status jenkins-agent
- sudo systemctl restart jenkins-agent
- journalctl -u jenkins-agent -n 50 --no-pager

## Result

The Jenkins pipeline completed successfully using the dedicated homelab inbound agent.

## Portfolio Value

This validates a real CI workflow using:

- Jenkins Pipeline from SCM
- GitHub repository integration
- Dedicated Jenkins inbound agent
- Homelab-based CI execution
- Systemd-managed agent service

## Latest Pipeline Enhancement

The Jenkins pipeline now includes YAML validation to ensure Kubernetes and configuration manifests are syntactically valid before changes are accepted.

Additional validation stages:

- YAML syntax validation
- Incident report directory validation
- Runbook directory validation
- Evidence directory validation

This improves the repository from basic CI validation into a more production-style documentation and manifest quality gate.
