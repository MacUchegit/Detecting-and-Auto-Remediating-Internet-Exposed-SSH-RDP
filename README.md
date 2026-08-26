# Detecting and Auto-Remediating Internet-Exposed SSH/RDP in AWS

**Project Type:** Cloud Security Engineering / Detection & Automated Remediation

**Company:** SanTechCorps

**Build Time:** ~3 hours

**Workload:** Amazon Linux 2023 on `t3.micro`

**AWS Services:** EC2, Security Groups, CloudTrail, EventBridge, Lambda, IAM, SNS, CloudWatch Logs

---

# 1. Executive Summary

I built an event-driven AWS security guardrail for SanTechCorps to address a common cloud-security problem: **accidental exposure of administrative services to the public internet**.

The protected workload was a live Amazon Linux EC2 web server.

Its approved network configuration allowed:

```text
HTTP 80 → 0.0.0.0/0
SSH 22  → Administrator-IP/32
```

During the security simulation, I deliberately introduced:

```text
SSH 22 → 0.0.0.0/0
```

This made SSH on the running EC2 workload publicly accessible.

Rather than relying on manual detection, I built an automated response pipeline that:

1. Recorded the Security Group modification with CloudTrail.
2. Detected `AuthorizeSecurityGroupIngress` using EventBridge.
3. Invoked a Python Lambda remediation function.
4. Inspected the affected Security Group.
5. Detected unrestricted SSH/RDP exposure.
6. Revoked only the offending rule.
7. Preserved legitimate application and administrator access.
8. Logged the remediation in CloudWatch.
9. Sent an incident notification through SNS.

The resulting workflow was:

```text
                  INTERNET
                      │
                   HTTP/80
                      │
                      ▼
          ┌──────────────────────┐
          │ SanTechCorps EC2     │
          │ Amazon Linux 2023    │
          │ t3.micro             │
          └──────────┬───────────┘
                     │
               Security Group
                     │
       ┌─────────────┴─────────────┐
       │                           │
HTTP 80 → 0.0.0.0/0       SSH 22 → Admin-IP/32
     APPROVED                    APPROVED
                                     │
                                     │ Engineer error
                                     ▼
                              SSH 22 → 0.0.0.0/0
                                   EXPOSED
                                     │
                                     ▼
                                CloudTrail
                                     │
                                     ▼
                                EventBridge
                                     │
                                     ▼
                                  Lambda
                              ┌──────┴──────┐
                              ▼             ▼
                        Revoke Rule        SNS
                              │             │
                              ▼             ▼
                       Secure State    Security Email
                              │
                              ▼
                       CloudWatch Logs
```

The final outcome was:

> **Internet-wide SSH access was automatically removed from the live EC2 workload while the approved administrator SSH rule and public web access remained operational.**

---

# 2. Business and Security Problem

SanTechCorps operates internet-facing workloads in AWS.

A legitimate web application may need to accept traffic from anywhere, but administrative services such as SSH and RDP should be tightly restricted.

The approved policy for this workload was:

| Service | Port | Source              | Status        |
| ------- | ---: | ------------------- | ------------- |
| HTTP    |   80 | `0.0.0.0/0`         | ✅ Approved    |
| SSH     |   22 | Administrator `/32` | ✅ Approved    |
| SSH     |   22 | `0.0.0.0/0`         | 🔴 Prohibited |
| RDP     | 3389 | `0.0.0.0/0`         | 🔴 Prohibited |
| SSH/RDP |    — | `::/0`              | 🔴 Prohibited |

The security control therefore needed to distinguish between:

```text
Internet-facing application traffic
              ✅

and

Internet-facing administrative access
              🔴
```

The hands-on demonstration used SSH/22 because the protected server ran Amazon Linux.

The same Lambda logic also protects RDP/3389 for Windows workloads.

---

# 3. Security Requirements

I designed the guardrail to:

* monitor inbound Security Group changes;
* identify the affected Security Group;
* operate only on resources explicitly enrolled for remediation;
* detect IPv4 `0.0.0.0/0`;
* detect IPv6 `::/0`;
* detect SSH/22;
* detect RDP/3389;
* identify ranges containing those ports;
* detect all-protocol/all-port exposure;
* revoke only the dangerous rule;
* preserve legitimate web access;
* preserve approved administrator SSH access;
* record the incident and remediation;
* notify the security team.

---

# 4. Resources Used

| Resource         | Name                                          |
| ---------------- | --------------------------------------------- |
| EC2 Instance     | `santechcorps-prod-web-01`                    |
| Security Group   | `santechcorps-prod-web-sg`                    |
| Protection Tag   | `AutoRemediateRemoteAccess=true`              |
| SNS Topic        | `santechcorps-security-alerts`                |
| CloudTrail Trail | `santechcorps-management-events`              |
| Lambda Function  | `santechcorps-remediate-public-remote-access` |
| IAM Policy       | `SanTechCorpsRemoteAccessRemediationPolicy`   |
| EventBridge Rule | `santechcorps-detect-public-remote-access`    |

---

# 5. Protected Workload Deployment

## Security Group Baseline

I created:

```text
santechcorps-prod-web-sg
```

in my selected VPC.

I configured the following inbound rules:

| Type | Port | Source        |
| ---- | ---: | ------------- |
| HTTP |   80 | Anywhere IPv4 |
| SSH  |   22 | My IP         |

The baseline therefore became:

```text
HTTP 80 → 0.0.0.0/0       ✅
SSH  22 → MY-IP/32         ✅
```

I deliberately allowed HTTP publicly because the instance represented an internet-facing web application.

SSH remained restricted to my administrator IP.

### 📸 Evidence 1 — Secure Firewall Baseline

I captured the Security Group inbound rules showing HTTP/80 publicly accessible and SSH/22 restricted to my `/32`.

**Caption:**

> **Figure 1 — SanTechCorps Security Group baseline permits public web traffic while restricting SSH administration to an approved administrator IP.**

---

## Remediation Enrollment Tag

I added the following tag to the Security Group:

```text
Key:   AutoRemediateRemoteAccess
Value: true
```

I used this tag as an authorization boundary for automated remediation.

The Lambda function could inspect Security Groups generally, but its IAM policy allowed rule revocation only where:

```text
AutoRemediateRemoteAccess=true
```

### 📸 Evidence 2 — Protection Tag

I captured the Security Group Tags tab showing:

```text
AutoRemediateRemoteAccess = true
```

**Caption:**

> **Figure 2 — Resource tagging explicitly enrolls the SanTechCorps Security Group in automated remote-access remediation.**

---

# 6. EC2 Workload

I launched a real EC2 workload using:

```text
Name:
santechcorps-prod-web-01

AMI:
Amazon Linux 2023

Instance type:
t3.micro

Public IPv4:
Enabled

Security Group:
santechcorps-prod-web-sg

Storage:
8 GiB gp3
```

I did not attach an IAM role to the instance because the application did not require AWS API access.

This reduced unnecessary permissions on the workload.

### 📸 Evidence 3 — Protected EC2 Workload

I captured the instance details showing:

* instance name;
* Running state;
* `t3.micro`;
* Amazon Linux;
* public IPv4;
* attached Security Group.

**Caption:**

> **Figure 3 — Live Amazon Linux EC2 workload deployed as the SanTechCorps application server protected by the monitored Security Group.**

---

# 7. Web Application Configuration

I connected to the instance using key-based SSH authentication:

```bash
ssh -i "santechcorps-lab-key.pem" ec2-user@PUBLIC_IP
```

I installed Apache:

```bash
sudo dnf install -y httpd
sudo systemctl enable --now httpd
```

I then created a lightweight demonstration application:

```bash
sudo tee /var/www/html/index.html > /dev/null <<'EOF'
<h1>SanTechCorps Production Web Server</h1>
<p>Cloud Security Automation Lab</p>
<p>Status: Operational</p>
EOF
```

I validated it through:

```text
http://PUBLIC_IP
```

This established that the Security Group was protecting an actual functioning workload rather than an unattached lab resource.

### 📸 Evidence 4 — Functional Application

I captured the SanTechCorps application page in the browser.

**Caption:**

> **Figure 4 — SanTechCorps web application running successfully on the protected Amazon Linux EC2 workload.**

---

# 8. Security Notification Channel

I created the SNS topic:

```text
santechcorps-security-alerts
```

and configured an email subscription.

I confirmed the subscription before continuing with the remediation workflow.

### 📸 Evidence 5 — SNS Configuration

I captured the subscription showing:

```text
Status: Confirmed
```

**Caption:**

> **Figure 5 — SNS notification channel configured to deliver automated remediation alerts to the SanTechCorps security team.**

---

# 9. CloudTrail Audit Logging

I configured CloudTrail to record the API activity responsible for Security Group changes.

Where an existing suitable trail was already present, it could be reused. Otherwise, I used:

```text
Trail:
santechcorps-management-events

Management events:
Enabled

API activity:
Write

Data events:
Disabled

Insights:
Disabled
```

I verified:

```text
Logging: ON
```

This enabled the project to answer:

```text
WHO changed the Security Group?

WHEN?

FROM WHAT SOURCE IP?

WHICH API ACTION WAS USED?

WHICH RESOURCE WAS MODIFIED?
```

### 📸 Evidence 6 — CloudTrail Configuration

I captured:

* trail name;
* Logging ON;
* management events enabled;
* Write activity enabled.

**Caption:**

> **Figure 6 — CloudTrail write-management event logging provides an auditable record of Security Group configuration changes.**

---

# 10. Lambda Remediation Function

I created:

```text
santechcorps-remediate-public-remote-access
```

using Python.

I configured:

```text
Memory:
128 MB

Timeout:
30 seconds
```

I also created the environment variable:

```text
SNS_TOPIC_ARN
```

containing the ARN of:

```text
santechcorps-security-alerts
```

### 📸 Evidence 7 — Lambda Configuration

I captured:

* Lambda name;
* Python runtime;
* memory;
* timeout;
* `SNS_TOPIC_ARN`.

**Caption:**

> **Figure 7 — Lambda remediation function configured with the SNS notification topic required for incident alerting.**

---

# 11. Least-Privilege IAM Design

Rather than granting the Lambda broad EC2 permissions, I restricted its authority.

The function could inspect Security Groups:

```text
ec2:DescribeSecurityGroups
ec2:DescribeSecurityGroupRules
```

but could revoke ingress only where the target Security Group had:

```text
AutoRemediateRemoteAccess=true
```

I used the following inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "InspectSecurityGroups",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSecurityGroupRules"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RemediateProtectedSecurityGroups",
      "Effect": "Allow",
      "Action": "ec2:RevokeSecurityGroupIngress",
      "Resource": "arn:aws:ec2:<REGION>:<ACCOUNT_ID>:security-group/*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/AutoRemediateRemoteAccess": "true"
        }
      }
    },
    {
      "Sid": "NotifySecurityTeam",
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "<SNS_TOPIC_ARN>"
    }
  ]
}
```

I named the policy:

```text
SanTechCorpsRemoteAccessRemediationPolicy
```

This design reduced the blast radius of the automation itself.

### 📸 Evidence 8 — Least-Privilege IAM

I captured the IAM policy showing:

```text
ec2:RevokeSecurityGroupIngress
```

and:

```text
ec2:ResourceTag/AutoRemediateRemoteAccess
```

**Caption:**

> **Figure 8 — Least-privilege IAM policy restricts automated firewall remediation to explicitly tagged Security Groups.**

---

# 12. Detection and Remediation Logic

I wrote a Python Lambda function that inspected the current Security Group configuration rather than blindly trusting the triggering event.

The function evaluated:

```text
0.0.0.0/0
::/0
```

and checked whether the public rule exposed:

```text
SSH 22
RDP 3389
port ranges containing 22/3389
all protocols/all ports
```

The core implementation was:

```python
import boto3
import json
import os

ec2 = boto3.client("ec2")
sns = boto3.client("sns")

SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]

TAG_KEY = "AutoRemediateRemoteAccess"
TAG_VALUE = "true"

PROTECTED_PORTS = {
    22: "SSH/22",
    3389: "RDP/3389"
}


def is_public(rule):
    return (
        rule.get("CidrIpv4") == "0.0.0.0/0"
        or rule.get("CidrIpv6") == "::/0"
    )


def exposed_services(rule):
    protocol = str(rule.get("IpProtocol"))

    if protocol == "-1":
        return list(PROTECTED_PORTS.values())

    if protocol != "tcp":
        return []

    start = rule.get("FromPort")
    end = rule.get("ToPort")

    if start is None or end is None:
        return []

    return [
        name
        for port, name in PROTECTED_PORTS.items()
        if start <= port <= end
    ]


def lambda_handler(event, context):

    detail = event.get("detail", {})

    if detail.get("eventName") != "AuthorizeSecurityGroupIngress":
        return {"status": "ignored"}

    group_id = (
        detail.get("requestParameters", {})
        .get("groupId")
    )

    if not group_id:
        raise ValueError("Security Group ID not found.")

    sg = ec2.describe_security_groups(
        GroupIds=[group_id]
    )["SecurityGroups"][0]

    tags = {
        tag["Key"]: tag["Value"]
        for tag in sg.get("Tags", [])
    }

    if tags.get(TAG_KEY) != TAG_VALUE:
        return {
            "status": "ignored",
            "reason": "Security Group not enrolled"
        }

    rules = ec2.describe_security_group_rules(
        Filters=[
            {
                "Name": "group-id",
                "Values": [group_id]
            }
        ]
    )["SecurityGroupRules"]

    dangerous = []

    for rule in rules:

        if rule.get("IsEgress"):
            continue

        if not is_public(rule):
            continue

        services = exposed_services(rule)

        if services:
            dangerous.append({
                "rule_id": rule["SecurityGroupRuleId"],
                "source": (
                    rule.get("CidrIpv4")
                    or rule.get("CidrIpv6")
                ),
                "services": services
            })

    if not dangerous:
        result = {
            "status": "no-threat-found",
            "group_id": group_id
        }

        print(json.dumps(result))
        return result

    ec2.revoke_security_group_ingress(
        GroupId=group_id,
        SecurityGroupRuleIds=[
            rule["rule_id"]
            for rule in dangerous
        ]
    )

    actor = (
        detail.get("userIdentity", {}).get("arn")
        or detail.get("userIdentity", {}).get("principalId")
        or "Unknown"
    )

    alert = {
        "severity": "HIGH",
        "security_group": group_id,
        "security_group_name": sg.get("GroupName"),
        "actor": actor,
        "source_ip": detail.get("sourceIPAddress"),
        "event_time": detail.get("eventTime"),
        "removed_rules": dangerous,
        "action": "Public SSH/RDP access revoked"
    }

    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject="SanTechCorps Alert: Public SSH/RDP Remediated",
        Message=json.dumps(alert, indent=2)
    )

    result = {
        "status": "remediated",
        "group_id": group_id,
        "removed_rules": dangerous
    }

    print(json.dumps(result))

    return result
```

The remediation flow was:

```text
Security Group changed
        ↓
Confirm resource is enrolled
        ↓
Retrieve current rules
        ↓
Detect world-open source
        ↓
Check SSH/RDP exposure
        ↓
Identify exact rule ID
        ↓
Revoke only dangerous rule
        ↓
Notify security
```

### 📸 Evidence 9 — Lambda Security Logic

I captured the Lambda code showing:

```python
is_public()
```

and:

```python
revoke_security_group_ingress()
```

**Caption:**

> **Figure 9 — Python remediation logic detects internet-wide SSH/RDP exposure and removes only the offending Security Group rule.**

---

# 13. Event-Driven Detection

I configured EventBridge to monitor CloudTrail for the API operation used when new Security Group ingress permissions are created.

I created:

```text
santechcorps-detect-public-remote-access
```

on the default event bus.

The event pattern was:

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["ec2.amazonaws.com"],
    "eventName": ["AuthorizeSecurityGroupIngress"]
  }
}
```

The target was:

```text
santechcorps-remediate-public-remote-access
```

This made the design event-driven rather than dependent on periodic polling.

### 📸 Evidence 10 — EventBridge Detection

I captured:

* rule name;
* Enabled state;
* event pattern;
* Lambda target.

**Caption:**

> **Figure 10 — EventBridge monitors CloudTrail for `AuthorizeSecurityGroupIngress` activity and invokes the automated remediation function.**

---

# 14. Secure Pre-Incident State

Before performing the security simulation, I verified:

```text
EC2 running                     ✅
Web application available       ✅
SSH restricted to my IP         ✅
Remediation tag present         ✅
CloudTrail logging              ✅
EventBridge enabled             ✅
Lambda deployed                 ✅
SNS subscription confirmed      ✅
```

I did not introduce public SSH access until the detection and remediation pipeline was ready.

---

# 15. Controlled Security Incident

I simulated the type of human error the control was designed to handle.

The existing approved rules were:

```text
HTTP 80 → 0.0.0.0/0
SSH 22 → MY-IP/32
```

I deliberately added a second SSH rule:

```text
SSH
TCP 22
0.0.0.0/0
```

The Security Group temporarily contained:

```text
HTTP 80 → 0.0.0.0/0       ✅ Approved

SSH 22 → MY-IP/32          ✅ Approved

SSH 22 → 0.0.0.0/0         🔴 Dangerous
```

Because the Security Group was attached to the public Amazon Linux EC2 instance and SSH was running, this represented a genuine remote-access exposure.

### 📸 Evidence 11 — Controlled Exposure

I captured the Security Group editor immediately before saving the rule.

The screenshot showed both:

```text
SSH 22 → MY-IP/32
SSH 22 → 0.0.0.0/0
```

**Caption:**

> **Figure 11 — Controlled SanTechCorps incident simulation introduces internet-wide SSH access to the live EC2 workload while retaining the approved administrator rule.**

I then saved the rule.

The resulting AWS activity was:

```text
AuthorizeSecurityGroupIngress
            ↓
CloudTrail
            ↓
EventBridge
            ↓
Lambda
            ↓
Security rule analysis
            ↓
RevokeSecurityGroupIngress
            ↓
SNS notification
```

---

# 16. Automated Containment

After the automation executed, I refreshed the Security Group.

The final state was:

```text
HTTP 80 → 0.0.0.0/0       ✅ Preserved

SSH 22 → MY-IP/32          ✅ Preserved

SSH 22 → 0.0.0.0/0         ❌ Removed
```

The control therefore removed the **unsafe condition**, rather than disabling SSH entirely.

### 📸 Evidence 12 — Automatic Remediation

I captured the Security Group after remediation.

**Caption:**

> **Figure 12 — Automated remediation removed internet-wide SSH access while preserving approved administrator and application traffic.**

---

# 17. Service Availability Validation

I verified that the SanTechCorps web application remained accessible after remediation.

I also opened a new SSH connection from my approved IP:

```bash
ssh -i "santechcorps-lab-key.pem" ec2-user@PUBLIC_IP
```

The connection remained successful.

The resulting security posture was:

```text
Public web users
HTTP/80
      ✅ Allowed

Approved administrator
SSH/22
      ✅ Allowed

Entire internet
SSH/22
      ❌ Blocked
```

This demonstrated that automated remediation did not unnecessarily disrupt legitimate access.

---

# 18. CloudTrail Incident Investigation

I used CloudTrail Event History to investigate the original modification.

I searched for:

```text
AuthorizeSecurityGroupIngress
```

and reviewed:

* actor identity;
* timestamp;
* source IP;
* AWS Region;
* Security Group ID;
* port 22;
* `0.0.0.0/0`.

This allowed me to determine:

```text
WHO made the change?
WHEN did it happen?
FROM WHERE?
WHAT API operation was used?
WHICH resource was affected?
```

### 📸 Evidence 13 — Incident Attribution

I captured the relevant CloudTrail event.

**Caption:**

> **Figure 13 — CloudTrail identifies the actor, source IP, timestamp and API request responsible for exposing SSH to the internet.**

---

# 19. Remediation Audit Trail

I also searched CloudTrail for:

```text
RevokeSecurityGroupIngress
```

This exposed the second side of the incident:

```text
Human action:
AuthorizeSecurityGroupIngress

          ↓

Automated response:
RevokeSecurityGroupIngress
```

The remediation event was associated with the Lambda execution identity rather than the user who introduced the insecure rule.

### 📸 Evidence 14 — Automated Response Audit

I captured the `RevokeSecurityGroupIngress` CloudTrail event.

**Caption:**

> **Figure 14 — CloudTrail records the automated `RevokeSecurityGroupIngress` action used to restore the approved Security Group configuration.**

---

# 20. CloudWatch Remediation Evidence

Lambda automatically wrote execution output to CloudWatch Logs.

I inspected the latest log stream and confirmed:

```text
"status": "remediated"
```

along with:

```text
"source": "0.0.0.0/0"
"services": ["SSH/22"]
```

### 📸 Evidence 15 — Lambda Execution

I captured the CloudWatch log showing the detection and remediation result.

**Caption:**

> **Figure 15 — CloudWatch logs confirm that Lambda detected and automatically removed the internet-exposed SSH rule.**

---

# 21. Security Team Notification

SNS delivered an alert containing contextual incident information including:

```text
Severity:
HIGH

Security Group:
santechcorps-prod-web-sg

Actor:
...

Source IP:
...

Removed Rule:
SSH/22
0.0.0.0/0

Action:
Public SSH/RDP access revoked
```

The automated remediation handled immediate containment while the alert provided information needed for follow-up investigation.

### 📸 Evidence 16 — SNS Incident Alert

I captured the SNS email and redacted unnecessary personal/account information.

**Caption:**

> **Figure 16 — SNS delivers contextual incident information to the SanTechCorps security team following automated containment.**

---

# 22. False-Positive Reduction

An important part of the project was demonstrating what the automation **did not remove**.

After the incident:

```text
HTTP 80 → 0.0.0.0/0
```

remained because public application access was intentional.

Similarly:

```text
SSH 22 → MY-IP/32
```

remained because restricted administrator access complied with policy.

Only:

```text
SSH 22 → 0.0.0.0/0
```

was revoked.

The final logic therefore behaved as:

```text
SSH 22 → 0.0.0.0/0       🔴 REMOVE

SSH 22 → ::/0             🔴 REMOVE

RDP 3389 → 0.0.0.0/0     🔴 REMOVE

SSH 22 → Admin-IP/32      ✅ PRESERVE

HTTP 80 → 0.0.0.0/0      ✅ PRESERVE
```

### 📸 Evidence 17 — Final Secure State

I captured the final inbound Security Group configuration showing:

* HTTP/80 publicly accessible;
* SSH/22 restricted to my `/32`;
* no internet-wide SSH rule.

**Caption:**

> **Figure 17 — Final security posture demonstrates false-positive reduction by preserving legitimate web and administrator access while preventing unrestricted SSH exposure.**

---

# 23. Incident Summary

**Incident:** Internet-Exposed SSH Access
**Severity:** High

**Affected Workload:**

```text
santechcorps-prod-web-01
```

**Affected Security Group:**

```text
santechcorps-prod-web-sg
```

**Unsafe Configuration:**

```text
SSH TCP/22 → 0.0.0.0/0
```

**Detection:**

```text
CloudTrail
     ↓
EventBridge
```

**Containment:**

```text
Lambda
     ↓
RevokeSecurityGroupIngress
```

**Notification:**

```text
SNS
```

**Outcome:**

> **Internet-wide SSH access was automatically removed while legitimate web traffic and administrator-specific SSH access remained operational.**

**Root Cause:**

> Human configuration error introduced an overly permissive inbound Security Group rule.

**Preventive Control:**

> Event-driven automated Security Group remediation using CloudTrail, EventBridge, Lambda, resource tagging and least-privilege IAM.

---

# 24. MITRE ATT&CK Context

The misconfiguration created an exposure that could make external remote administration services available to attackers.

Relevant concepts include:

```text
T1133
External Remote Services

T1021.004
SSH

T1021.001
Remote Desktop Protocol
```

The project simulated the **cloud configuration condition** that could enable abuse of these services.

It did not attempt credential attacks or compromise the server.

---

# 25. Key Security Engineering Decisions

## Event-Driven Detection

I used EventBridge instead of periodically polling Security Groups.

```text
Configuration change
        ↓
Security evaluation
```

This reduced detection delay and unnecessary API calls.

## Scoped Automated Remediation

I used:

```text
AutoRemediateRemoteAccess=true
```

to restrict which Security Groups the Lambda could modify.

Automation therefore had defined authority rather than unrestricted control.

## Precise Rule Removal

The Lambda revoked the exact offending Security Group Rule ID.

It did not remove:

* all SSH access;
* all internet access;
* the entire Security Group.

## Human Notification After Containment

Lambda contained the immediate exposure automatically.

SNS then gave security personnel the information necessary for investigation.

---

# 26. Skills Demonstrated

The project demonstrated practical experience with:

* Amazon EC2
* Amazon Linux
* Security Groups
* secure remote administration
* IAM least privilege
* CloudTrail auditing
* EventBridge
* Python
* AWS Lambda
* automated remediation
* CloudWatch Logs
* Amazon SNS
* incident investigation
* false-positive reduction
* cloud security governance

---

# 27. Evidence Summary

| Figure | Evidence                       | Purpose                   |
| -----: | ------------------------------ | ------------------------- |
|      1 | Secure Security Group baseline | Approved network posture  |
|      2 | Remediation tag                | Automation scope          |
|      3 | Running EC2 workload           | Real protected resource   |
|      4 | Functional application         | Workload validation       |
|      5 | SNS subscription               | Notification channel      |
|      6 | CloudTrail configuration       | Audit logging             |
|      7 | Lambda configuration           | Remediation service       |
|      8 | IAM policy                     | Least privilege           |
|      9 | Lambda code                    | Custom detection logic    |
|     10 | EventBridge rule               | Event-driven detection    |
|     11 | Public SSH exposure            | Incident simulation       |
|     12 | Remediated Security Group      | Automated containment     |
|     13 | CloudTrail authorization event | Incident attribution      |
|     14 | CloudTrail revoke event        | Response audit trail      |
|     15 | CloudWatch Lambda logs         | Execution evidence        |
|     16 | SNS notification               | Human alerting            |
|     17 | Final secure state             | False-positive validation |

Together, the evidence tells the complete story:

```text
REAL WORKLOAD
      ↓
SECURE BASELINE
      ↓
SECURITY MISCONFIGURATION
      ↓
DETECTION
      ↓
AUTOMATED CONTAINMENT
      ↓
AUDIT INVESTIGATION
      ↓
SECURITY NOTIFICATION
      ↓
VALIDATED SECURE STATE
```

---

# 28. Cost Management and Cleanup

I intentionally kept the architecture small.

The lab did not require:

```text
NAT Gateway
Load Balancer
RDS
Traffic Mirroring
Auto Scaling
```

I used only one small EC2 instance and terminated resources immediately after completing the validation.

After collecting all evidence, I:

1. Terminated `santechcorps-prod-web-01`.
2. Deleted `santechcorps-detect-public-remote-access`.
3. Deleted `santechcorps-remediate-public-remote-access`.
4. Deleted its CloudWatch log group.
5. Deleted `santechcorps-security-alerts`.
6. Deleted `santechcorps-prod-web-sg`.
7. Deleted the temporary EC2 key pair.
8. Deleted the CloudTrail trail/S3 bucket only if they were created specifically for the lab.
9. Preserved any pre-existing shared CloudTrail trail or default VPC.

### 📸 Evidence 18 — Resource Cleanup

I captured the terminated EC2 instance or cleaned-up resource state.

**Caption:**

> **Figure 18 — SanTechCorps lab resources were removed after validation to maintain cost-efficient AWS resource management.**

---

# 29. Final Project Result

The project demonstrated a complete Cloud Security Engineering lifecycle:

```text
Real AWS workload
        ↓
Secure configuration
        ↓
Human configuration error
        ↓
CloudTrail visibility
        ↓
EventBridge detection
        ↓
Lambda decision
        ↓
Automated containment
        ↓
CloudWatch evidence
        ↓
SNS notification
        ↓
Incident investigation
        ↓
Secure final state
```

Rather than simply alerting on a dangerous configuration, the control automatically restored the approved security posture.

---

# 30. Recruiter-Friendly Project Summary

## Automated Detection & Remediation of Internet-Exposed SSH/RDP in AWS

> **I built an event-driven AWS security guardrail protecting a live Amazon Linux EC2 workload. CloudTrail and EventBridge detected unrestricted administrative access, while a least-privilege Python Lambda automatically revoked dangerous SSH/RDP Security Group rules, preserved approved application and administrator connectivity, recorded remediation evidence in CloudWatch, and alerted the security team through SNS.**

---

# 31. CV Bullet

> **Built an event-driven AWS security guardrail that detected internet-exposed SSH/RDP access on a live EC2 workload through CloudTrail and EventBridge, automatically revoked dangerous ingress rules using a least-privilege Python Lambda function, and delivered contextual incident notifications through SNS while preserving approved application and administrator traffic.**

---

# 32. Interview Explanation

> **“I deployed a public Amazon Linux EC2 workload representing a SanTechCorps application. Its Security Group allowed HTTP publicly while SSH was restricted to my administrator IP. I simulated a realistic configuration mistake by adding an additional SSH rule allowing TCP/22 from `0.0.0.0/0`, which genuinely exposed SSH on the running workload. CloudTrail recorded the `AuthorizeSecurityGroupIngress` action, and EventBridge invoked my Python Lambda remediation function. The function first verified that the Security Group was tagged for automated remediation, inspected its current rules, identified the unrestricted SSH rule and revoked its exact Security Group Rule ID. It preserved the legitimate `/32` SSH rule and public web access, wrote remediation evidence to CloudWatch Logs and sent an SNS alert containing the actor, source IP and affected Security Group. I then correlated the original authorization event with the automated `RevokeSecurityGroupIngress` event in CloudTrail to validate the complete response workflow.”**

---

# 33. Why This Project Stands Out

The architecture remained intentionally simple:

```text
EC2
 ↓
Security Group
 ↓
CloudTrail
 ↓
EventBridge
 ↓
Lambda
 ├── Remediation
 └── SNS
```

But the project solved a realistic security problem and demonstrated:

> **A real workload → a real cloud misconfiguration → automated detection → precise containment → investigation → alerting → validated recovery.**

The value of the project comes from the security engineering decision-making and evidence, not unnecessary architectural complexity.
