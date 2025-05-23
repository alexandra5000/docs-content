---
mapped_pages:
  - https://www.elastic.co/guide/en/security/current/vuln-management-faq.html
  - https://www.elastic.co/guide/en/serverless/current/security-vuln-management-faq.html
applies_to:
  stack: all
  serverless:
    security: all
products:
  - id: security
  - id: cloud-serverless
---

# Frequently asked questions (FAQ)

Frequently asked questions about the Cloud Native Vulnerability Management (CNVM) integration and features.

**Which security data sources does the CNVM integration use to identify vulnerabilities?**

The CNVM integration uses various security data sources. The complete list can be found [here](https://github.com/aquasecurity/trivy/blob/v0.35.0/docs/docs/vulnerability/detection/data-source.md).

**What’s the underlying scanner used by CNVM integration?**

CNVM uses the open source scanner [Trivy](https://github.com/aquasecurity/trivy) v0.35.

**What system architectures are supported?**

Because of Trivy’s limitations, CNVM can only be deployed on ARM-based VMs. However, it can scan hosts regardless of system architecture.

**How often are the security data sources synchronized?**

The CNVM integration fetches the latest data sources at the beginning of every scan cycle to ensure up-to-date vulnerability information.

**What happens if a scan cycle does not complete within 24 hours?**

If a scan cycle doesn’t finish within 24 hours, the ongoing cycle will continue until completion. When it finishes, a new cycle will immediately start.

**How is the lifecycle of snapshots handled?**

The CNVM integration manages the lifecycle of snapshots. Snapshots are automatically deleted/removed at the end of each scan cycle.

**Does CNVM have an impact on the user’s cloud expenses?**

Yes, CNVM creates additional cloud expenses, since scanning involves provisioning a new virtual machine to conduct the scan.

**Does CNVM also scan the new AWS EC2 instances that it creates?**

Yes, CNVM scans all AWS EC2 instances in every scan cycle, including any created by the integration.

**Does CNVM scan AWS EC2 instances with encrypted volumes?**

Encrypted volumes can be scanned only if they were encrypted using Amazon’s default EBS key, and are *not* running Amazon Linux 2023.

**Does CNVM prevent multiple installations in a single region?**

No, CNVM does not currently prevent redundant deployment to the same region.

**What volume types and file systems does CNVM support?**

CNVM supports all AWS EBS volume types and works with `ext4` and `xfs` file systems.

**Does CNVM scan stopped EC2 instances?**

Yes, CNVM scans all EC2 instances, whether they are running or stopped, to ensure comprehensive vulnerability detection.

**What AWS permissions does the user require to run the CloudFormation template for CNVM onboarding?**

To run the CloudFormation template for CNVM onboarding, you need an AWS user account with permissions to perform the following actions: run CloudFormation templates, create IAM Roles and InstanceProfiles, and create EC2 SecurityGroups and Instances.

**Why do I get an error when I try to run the CloudFormation template?**

It’s possible you’re using an unsupported region. Currently the `eu-north-1` and `af-south-1` regions are not supported because they don’t provide the required instance types.

