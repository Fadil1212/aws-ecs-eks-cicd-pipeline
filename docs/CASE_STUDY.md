# Engineering case study: one immutable release across ECS and EKS

**Project:** Container delivery with GitHub, Jenkins and AWS  
**Date of validated lab:** November 13, 2026  
**Release:** `2.0.0`  
**Author:** Fadil Mohammed Surur  
**Outcome:** Working end-to-end pipeline, verified on ECS and EKS, followed by controlled recovery/scaling tests and teardown.

## Problem

A release appearing healthy in two environments does not prove both are running **the same artifact**. Different builds, tags and manual deployment steps can quietly introduce drift. I used a reproducible pipeline that identifies one source revision, builds one image, deploys it by immutable digest and verifies the resulting services independently.

## Design choices

- **One application, two runtimes:** a small Python HTTP server avoids application differences when comparing orchestration.
- **Two layers of CI:** GitHub Actions gates pull requests; Jenkins performs the deployment and publishes verification artifacts.
- **Immutable image identity:** a digest is stronger evidence than a mutable `latest` tag.
- **Health probes, not only process status:** `/healthz` exposes an intentional failure switch for reproducible recovery testing.
- **Private application access:** the demo was checked through local SSM tunnels rather than exposing a public production URL.
- **Infrastructure as code:** CloudFormation set up the workshop resources, including ECR, ECS, EKS, Jenkins, network security and an S3 audit bucket.

## Release timeline

| Milestone | Evidence |
| --- | --- |
| Prerequisites installed | Python, Git, AWS CLI, SSM plugin and kubectl checks passed |
| Repository initialized | Main branch with GitHub Actions workflow |
| First pipeline | Jenkins Build #1 completed successfully |
| Feature branch | `enhancement/platform-comparison` added `/api/compare`, release 2.0 and a regression test |
| CI validation | Five Python tests passed; GitHub PR #1 passed both checks |
| Merge | Commit `4d6c5f387c0f468d3104218ecd09d866ca63ddf2` |
| Deploy | ECS and EKS both reported version `2.0.0`, healthy response and same source prefix |
| Verify | `verification.json` had `status: PASS` and matching image digest |
| Recovery extension | Jenkins Build #4 `SUCCESS`, with controlled ECS/EKS failures and recovery |
| Cleanup | ECS service deleted; workshop CloudFormation stack `DELETE_COMPLETE`; temporary GitHub key removed; Jenkins secret scheduled for deletion |

## What the recovery exercise demonstrated

**EKS:** The controlled change made the replacement Kubernetes rollout unhealthy. The experiment captured rollout failure, performed `kubectl rollout undo`, and observed a successful return to the previous healthy configuration.

**ECS:** The controlled change caused replacement tasks to fail container health checks. With the service deployment circuit breaker configured, ECS rolled back to the earlier working task definition. This mechanism differed from the explicit EKS undo used in the exercise.

**Scaling:** The same app ran at two ECS tasks and two EKS ready replicas. The automation restored the desired count to one on each platform and the final verification passed.

This is a **functional demonstration**. It does not establish which platform has better throughput, reliability in production, lower long-term total cost or better autoscaling under load.

## Release traceability

```json
{
  "status": "PASS",
  "version": "2.0.0",
  "source_commit": "4d6c5f387c0f468d3104218ecd09d866ca63ddf2",
  "container_digest": "sha256:47415236fabbd4df068d59d4427eaac05218bbeff72ce479b2bfe0f81c8b897c",
  "ecs_platform": "ECS",
  "eks_platform": "EKS",
  "final_health": "PASS"
}
```

*Above is a **readable summary** of fields confirmed from workshop artifacts, not a verbatim copy of the original JSON schema.*

## ECS vs EKS: engineering trade-offs

| Dimension | ECS on Fargate | EKS on managed EC2 |
| --- | --- | --- |
| Workload configuration | Task definition + ECS service | Kubernetes Deployment + Service |
| Worker management | Fargate abstracts worker hosts | Managed EC2 node group provides worker capacity |
| Scheduling | ECS service scheduler | Kubernetes scheduler |
| Failure recovery in this lab | Circuit breaker automatic rollback | Explicit rollout undo after failed deployment |
| Replicas | Desired task count | Deployment replica count |
| Best fit in this comparison | Simple AWS-centric API with a small operations footprint | Kubernetes-native deployments and workflows |

## Security and resource lifecycle

The public repository contains **no AWS credentials, Jenkins passwords, API tokens or kubeconfig**. The workshop used read-only GitHub deploy access for Jenkins and SSM port forwarding for private previews. The audit object used S3-managed server-side encryption (SSE-S3). After saving results, the ECS service was deleted, the S3 audit bucket emptied, the CloudFormation stack removed, the GitHub deploy key deleted, and the Jenkins secret scheduled for deletion. The separate AWS Academy infrastructure stack was intentionally retained.

## Lessons learned

1. Separate **code correctness** (tests) from **release correctness** (commit and digest verification).
2. Put failure recovery into a reproducible experiment rather than relying on a green health badge.
3. Understand that Kubernetes rollout undo and ECS's configured circuit breaker are different behaviors.
4. When a localhost preview fails, investigate the **SSM tunnel** separately from application health.
5. Treat infrastructure teardown as part of the delivery lifecycle — ECR repositories may block CloudFormation deletion while images remain.

## Evidence and references

- [Screenshots and evidence](README.md)
- [Merged feature PR](https://github.com/Fadil1212/student-starter/pull/1)
- [GitHub Actions runs](https://github.com/Fadil1212/student-starter/actions)
- [Jenkins pipeline definition](../Jenkinsfile)
- [Recovery extension implementation](../scripts/cloud_extension.py)
- [Workshop report](ECS_EKS_CICD_Project_Report.docx)

**Provenance note:** Original AWS lab resources no longer exist. Screenshots and the report are historical evidence from the completed November 13 workshop, not live endpoints.
