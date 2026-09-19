---
title: "Week 6 - CloudWatch Monitoring, Project Packaging, and Final Cloud File Manager Report"
weight: 6
chapter: false
pre: "<b>1.6. </b>"
---

# WORKLOG WEEK 6

**Date range:** 07/09/2026 - 13/09/2026

## Week 6 Objectives

- Study Amazon CloudWatch and AWS CloudTrail for monitoring cloud-native applications.
- Configure logs, metrics, and alarms for the API server, SQS queue, and Durable Worker processes.
- Set up automated 75 USD CloudWatch Billing Alarms integrated with Amazon SNS email notifications.
- Consolidate all acquired AWS competencies, completing end-to-end verification and packaging for **Cloud File Manager**.
- Author the comprehensive hands-on Workshop guide and finalize the capstone Internship Report.

## Tasks Completed During the Week

| Day | Task | Start Date | Completion Date | Reference |
| --- | --- | --- | --- | --- |
| Mon - Tue | Study Amazon CloudWatch, CloudTrail, and integrate structured logging for the Cloud File Manager API and Worker | 07/09/2026 | 08/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| Wed | Configure CloudWatch Metrics, Alarms, and set up the 75 USD Billing Alarm with SNS Email Alerts | 09/09/2026 | 09/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| Thu | Test Dead-Letter Queue (SQS) error recovery and verify Lease Fencing under concurrent worker workloads | 10/09/2026 | 10/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| Fri - Sat | Finalize comprehensive hands-on documentation for the Cloud File Manager Workshop (Modules 5.1 through 5.7) | 11/09/2026 | 12/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| Sun | Package all source code artifacts, Terraform configurations, and complete the final Internship Report | 13/09/2026 | 13/09/2026 | — |

### System implementation scope

- **Cloud File Manager** cloud storage and management system.
- Backend using Node.js, Express.js, and Sequelize ORM (MySQL).
- Frontend built with React 18, TypeScript, Vite, and Ant Design 5.
- Direct-to-S3 uploads via Presigned POST Policies coupled with atomic Quota Reservations.
- Serverless AWS Lambda validating binary magic bytes triggered automatically by S3 events.
- Autonomous Durable Worker for asynchronous processing, database-backed Lease Fencing, and Jittered Exponential Backoff retries.
- File lifecycle management, Trash, version-aware S3 permanent purges, and automated self-healing Quota Reconciliation.

### Proposed AWS architecture

```text
Client (React 18 + Vite)
→ Backend API (Express + Sequelize) & Amazon S3 (Direct Upload)
→ AWS Lambda (Magic Bytes Validator) & Amazon SQS (Dead-Letter Queue)
→ MySQL Database & Durable Background Worker
→ Amazon CloudWatch (Logs, Metrics, Billing Alarms)
```

The system architecture also integrates Amazon S3, Amazon VPC, Security Groups, AWS IAM, AWS Systems Manager Parameter Store, Terraform IaC, and GitHub Actions CI/CD.

## Week 6 Results

- Successfully applied Amazon CloudWatch and AWS CloudTrail to production-grade operational monitoring.
- Established a complete operational observability suite with Logs, Metrics, and Alarms across Cloud File Manager.
- Configured the 75 USD Billing Alarm ensuring safe, controlled cloud infrastructure costs.
- Fully packaged the Cloud File Manager project, achieving passing results across 82 backend tests, 6 frontend tests, and Terraform HCL security checks.
- Completed the comprehensive hands-on Workshop guide and finalized the Internship Report according to the milestone schedule.
