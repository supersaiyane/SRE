# SRE / DevOps / Platform Engineering — Study & Reference Guide

> A comprehensive, hands-on knowledge base covering Site Reliability Engineering, DevOps practices, and Platform Engineering. Built for learning, interview prep, and day-to-day reference.

---

## How to Use This Repo

- **Learning**: Follow the numbered files in `SRE/` from 1 to 10 for a structured SRE curriculum.
- **Interview Prep**: Each section includes interview-style questions and scenario walkthroughs.
- **Hands-On Practice**: The `Demo_Apps/` directory contains 12 working demo applications with full configs.
- **Quick Reference**: Use this README as your table of contents to jump to any topic.

---

## Table of Contents

### Core SRE Fundamentals (`SRE/`)

| # | Topic | What You'll Learn |
|---|-------|-------------------|
| 1 | [Basic Terminology](SRE/1_Basic_Terminology.md) | SLA, SLO, SLI, Error Budget, Golden Signals, Incident Metrics, Deployment Strategies |
| 2 | [Incident Lifecycle](SRE/2_Incident%20Lifecycle%20-%20Stage-by-Stage.md) | 9-stage incident response from detection to closure |
| 3 | [SLI to SLO to SLA Process](SRE/3_Process.md) | Step-by-step process for defining reliability targets |
| 4 | [Golden Signals Monitoring](SRE/4_GoldenSignals.md) | Implementing Latency, Traffic, Errors, Saturation with Prometheus |
| 5 | [Grafana Dashboard Setup](SRE/5_Golden-SignalGrafanadashboard.md) | Building golden signal dashboards with PromQL |
| 6 | [SLI, SLO, SLA & Error Budget](SRE/6_SLI_SLO_SLA_ErrorBudget_Explained.md) | Nines, burn rates, PromQL queries for SLO monitoring |
| 7 | [Checkout API SLO Alerting](SRE/7_Checkout_API_SLO_Alerting_Setup.md) | Full Prometheus + Alertmanager + PagerDuty setup |
| 8 | [Error Budget Policies](SRE/8_error_budget_policy.md) | Multi-tier burn rate thresholds and response policies |
| 9 | [Error Budget Deep Dive](SRE/9_error_budget.md) | Measurement, alerting, corner cases, and best practices |
| 10 | [Toil](SRE/10_toil.md) | Identifying, prioritizing, and automating away toil |
| 11 | [On-Call Best Practices](SRE/11_oncall_best_practices.md) | Rotations, escalations, alert hygiene, burnout prevention |
| 12 | [Capacity Planning](SRE/12_capacity_planning.md) | Forecasting, load testing, scaling strategies |
| 13 | [Postmortem Template](SRE/13_postmortem_template.md) | Blameless postmortem structure with worked example |
| 14 | [Kubernetes Troubleshooting](SRE/14_kubernetes_troubleshooting.md) | Decision trees for CrashLoopBackOff, OOMKilled, Pending, networking issues |
| 15 | [PromQL Cookbook](SRE/15_promql_cookbook.md) | Copy-paste-ready queries for golden signals, SLOs, K8s health, databases |
| 16 | [Reliability Patterns](SRE/16_reliability_patterns.md) | Circuit breakers, retries, bulkheads, backpressure, rate limiting, fallbacks |
| 17 | [SRE Interview Mega Guide](SRE/17_sre_interview_guide.md) | 17 scenario-based questions with frameworks, scoring rubrics, STAR examples |

---

### Observability (`Observability/`)

| Topic | What You'll Learn |
|-------|-------------------|
| [End-to-End Observability Interview](Observability/End_to_End_Observability_Interview.md) | Designing observability from browser to database |

---

### Chaos Engineering (`chaosEngineering/`)

| Topic | What You'll Learn |
|-------|-------------------|
| [Complete Guide](chaosEngineering/Chaos_Engineering_Complete_Guide.md) | Principles, scenarios, tools (LitmusChaos, Gremlin, AWS FIS) |
| [Architecture Diagram](chaosEngineering/Chaos_Architecture_Diagram.md) | Visual architecture and experiment flow |
| [Interview Questions](chaosEngineering/Chaos_Engineering_Interview_Question.md) | Mock interview with detailed answer guide |
| [PromQL Labs](chaosEngineering/PromQL_Labs_for_Chaos_Observability.md) | PromQL queries for chaos experiment observability |

---

### Service Mesh (`ServiceMesh/`)

| Topic | What You'll Learn |
|-------|-------------------|
| [Service Mesh Deep Dive](ServiceMesh/Service_Mesh_Deep_Dive.md) | Istio, Linkerd, Consul, AWS App Mesh architecture |
| [Kiali SLO Dashboard](ServiceMesh/Kiali_SLO_Dashboard_and_Metrics.md) | Dashboard setup and SLO metrics with Kiali |
| [Case Study & Auto-Remediation](ServiceMesh/Mesh_Case_Study_and_AutoRemediation.md) | Real-world case studies and self-healing patterns |
| [Demo Bundle](ServiceMesh/service_mesh_demo_bundle/) | VirtualService, DestinationRule, PeerAuthentication YAMLs |

---

### Real User Monitoring (`RUM/`)

| Topic | What You'll Learn |
|-------|-------------------|
| [RUM Deep Dive](RUM/11_RUM_Deep_Dive.md) | RUM concepts, metrics, architecture, implementation |
| [RUM SLO Burn Tracking](RUM/RUM_SLO_Burn_Tracking.md) | SLO burn tracking with client-side data |

---

### Multi-Region & Disaster Recovery (`Multi-Region Failover/`)

| Topic | What You'll Learn |
|-------|-------------------|
| [Failover Architecture](Multi-Region%20Failover/Multi_Region_Failover_Architecture.md) | Route53, geo-routing, Prometheus federation, Thanos |
| [DR Runbook](Multi-Region%20Failover/Region_Failover_DR_Runbook.md) | Step-by-step region promotion procedure |
| [Interview Scenario](Multi-Region%20Failover/Multi_Region_Interview_Scenario.md) | Mock interview for multi-region design |

---

### GitOps & Progressive Delivery (`GitOps/`)

| Topic | What You'll Learn |
|-------|-------------------|
| [Argo Rollouts & Progressive Delivery](GitOps/Argo_Rollouts_GitOps_Progressive_Delivery.md) | Canary, blue-green, AnalysisTemplates, automated rollback |

---

### Kubernetes YAML Templates (`Learn_yamls/`)

| Topic | What You'll Learn |
|-------|-------------------|
| [YAML Fundamentals](Learn_yamls/yaml.md) | YAML syntax, data types, anchors, multi-doc files |
| [K8s YAML Templates](Learn_yamls/k8s_yaml_templates.md) | Pod, Deployment, Service, ConfigMap, Secret, HPA templates |

---

### Career Roadmaps (`RoadMaps/`)

| Roadmap | Target Role |
|---------|-------------|
| [SRE Architect Roadmap](RoadMaps/SRE_Architect_Roadmap.md) | SRE Practitioner to SRE Architect (4 phases) |
| [Platform Engineering Roadmap](RoadMaps/Platform_Engineering_Roadmap.md) | Platform Engineer to Platform Architect |
| [DevOps Roadmap](RoadMaps/DevOps_Roadmap.md) | DevOps Engineer to DevOps Architect |

---

### Demo Applications (`Demo_Apps/`)

12 fully working demo applications with Docker, Kubernetes, Helm, and Terraform configs:

| App | Stack | What It Demonstrates |
|-----|-------|----------------------|
| **golden-signals-calculator** | Node.js, Docker, K8s | Golden signals instrumentation with prom-client |
| **Complete_SRE_DEMO** | Node.js, Prometheus, Grafana, Helm | Full SRE stack: monitoring, alerting, SLO rules |
| **argo_rollouts_demo** | Node.js, Argo Rollouts | Canary deployments with AnalysisTemplates |
| **chaos_demo_app** | Node.js, LitmusChaos | Chaos engineering experiment with SLO tracking |
| **istio_autoheal_demo** | Node.js, Istio, Argo Workflows | Self-healing with service mesh + workflow engine |
| **rum_demo_app** | Node.js, HTML frontend | Real User Monitoring implementation |
| **rum_prometheus_bundle** | Node.js, Prometheus, Grafana | RUM metrics with HPA auto-scaling |
| **golden-signal-calc-helm** | Helm | Multi-environment Helm chart (dev/stage/prod) |
| **gitops_aws-main** | Terraform, K8s, AWS | Full GitOps pipeline on EKS with 30 TF files |
| **CICD-with-Docker-Compose** | Docker Compose, AWS CodeDeploy | CI/CD with Docker Compose |
| **CustomizedAws_DevOps** | Jenkins, AWS CodeBuild | Jenkins + AWS DevOps integration |
| **EFSfilesystem-in-K8s** | K8s, EFS, Kafka | EFS storage with Kafka/Zookeeper on K8s |

---

### Additional Resources

| Resource | Description |
|----------|-------------|
| [Platform Kit Automation](platform_kit_automation.md) | Interview reference for platform automation tooling |
| [LLM RAG Article](LLM_RAG_Article.md) | Building a private RAG assistant with local LLMs |
| [Ansible Playbooks](ansible%20playbooks.pdf) | Ansible automation reference (PDF) |
| [K8s Logging & Monitoring](k8s%20logging_monitoring.pdf) | Kubernetes observability guide (PDF) |
| [K8s Network Policies](network%20policies%20k8s.pdf) | Network policy reference (PDF) |
| [Tech Concepts](tech_concepts_vvvimp.pdf) | Important technical concepts (PDF) |

---

## Suggested Learning Path

```
Week 1-2:  SRE Fundamentals (files 1-6) + YAML Templates
Week 3:    Golden Signals + Grafana + Error Budgets (files 7-9)
Week 4:    Toil + On-Call + Capacity Planning (files 10-12)
Week 5:    Postmortem Template + Kubernetes Troubleshooting (files 13-14)
Week 6:    PromQL Cookbook + Reliability Patterns (files 15-16)
Week 7:    Chaos Engineering (all 4 files) + run chaos_demo_app
Week 8:    Service Mesh (all files) + run istio_autoheal_demo
Week 9:    GitOps + Progressive Delivery + run argo_rollouts_demo
Week 10:   Multi-Region + DR Runbook + Observability
Week 11:   RUM + run rum_demo_app
Week 12:   Full SRE demo (Complete_SRE_DEMO) + Interview Mega Guide (file 17)
```

---

## Contributing

Found something that could be improved? Open a PR or issue. Contributions welcome for:
- New topics (FinOps, AIOps, Platform Engineering deep dives)
- Additional demo applications
- More interview questions and scenarios
- Corrections and updates

---

## License

This repository is for educational purposes. Feel free to fork and use for your own learning.
