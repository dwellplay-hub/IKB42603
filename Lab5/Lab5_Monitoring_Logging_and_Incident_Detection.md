# Lab 5: Monitoring, Logging and Incident Detection



- Course Code: IKB42603
- Course Title: Cloud Computing Security Essentials
- Lab Title: Monitoring, Logging and Incident Detection
- Student Name: Muhammad Danish Isyraq
- Date: September 2026
- Environment: LocalStack, Docker, AWS CLI, CloudWatch Logs endpoint `http://localhost:4566`, Alpine Linux with `iptables`, SHA-256 hash-chaining utilities

## Lab Summary

This lab demonstrates how telemetry and log integrity are used to detect, analyse, and respond to a cloud security incident. The scenario models an authentication service that records repeated failed logins from a remote IP, followed by a successful login and a large data export. The activity is first centralised to LocalStack CloudWatch Logs, then inspected with log queries, protected with a hash chain to confirm tamper evidence, and correlated into a single incident alert.

The practical objective is to show that raw logs alone are not enough: they must be collected centrally, preserved securely, and correlated across time and identity to reveal malicious behaviour. The lab also demonstrates the incident response lifecycle through containment, evidence preservation, and forensic verification.

## Learning Outcomes

By completing this lab, students will be able to:

- Demonstrate CLO2 principles for cloud telemetry monitoring and analysis.
- Generate and centralise application logs into a managed log backend.
- Recognise security-relevant login patterns using `grep`, `awk`, and aggregation queries.
- Protect audit records using a tamper-evident SHA-256 hash chain.
- Correlate related events to identify a likely brute-force compromise and exfiltration.
- Apply the incident response lifecycle: detection, analysis, containment, evidence handling, and documentation.

## Evidence Mapping Table

| Evidence Artifact | Mapping to Task / Verification | Description |
|---|---|---|
| `Evidence/Task 1.png` | Task 1 | Authentication log generation showing the brute-force sequence and `EXPORT_DATA` event |
| `Evidence/Task 2.png` | Task 2 | Log shipping to LocalStack CloudWatch Logs using `put-log-events` |
| `Evidence/Task 3.png` | Task 3 | Query output identifying failed login attempts grouped by source IP |
| `Evidence/Task 4.png` | Task 4 | Hash-chained log chain with tampered log showing divergence |
| `Evidence/Task 5.png` | Task 5 | Correlation alert demonstrating likely brute-force → compromise → exfiltration |
| `Evidence/Task 6.png` | Task 6 | Firewall blocking rule and evidence hash collection |
| `Evidence/Verification Commands.png` | Verification Commands | LocalStack log group listing and integrity check evidence |

## Environment Setup

### LocalStack Container Deployment

The lab uses LocalStack to emulate AWS CloudWatch Logs locally.

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack
EP='--endpoint-url=http://localhost:4566'
```

### Create Log Group and Log Stream

```bash
aws $EP logs create-log-group --log-group-name /ccse/app
aws $EP logs create-log-stream --log-group-name /ccse/app --log-stream-name auth
```

This creates the central log group `/ccse/app` and the application stream `auth` for authentication telemetry.

### Evidence Screenshot

![LocalStack and CloudWatch Logs setup](Evidence/Task%201.png)

> Note: The screenshot referenced here maps the LocalStack setup and log storage configuration for the lab. The evidence folder contains the required task screenshots and verification screenshot.

---

# Session A: Logging and Centralisation

## Task 1: Generate Application Logs

The authentication service was simulated by creating `auth.log` with the suspicious sequence of events.

```bash
cat > auth.log <<'EOF'
2025-03-01T09:00:01 LOGIN_OK user=ahmad ip=10.0.0.5
2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:12 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:15 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:18 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:22 LOGIN_OK user=admin ip=203.0.113.9
2025-03-01T09:01:40 EXPORT_DATA user=admin ip=203.0.113.9 size=500MB
EOF
cat auth.log
```

### Interpretation of the Event Sequence

- `LOGIN_OK user=ahmad ip=10.0.0.5` indicates a legitimate successful login from a trusted address.
- `LOGIN_FAIL` entries against `admin` from `203.0.113.9` indicate repeated password guessing or brute-force behaviour.
- `LOGIN_OK user=admin ip=203.0.113.9` indicates that the attacker successfully completed authentication.
- `EXPORT_DATA` of `500MB` suggests likely data exfiltration from the compromised account.

This sequence is consistent with a multi-stage intrusion: brute-force access, successful compromise, then unauthorised export.

### Evidence Screenshot

![Application authentication log](Evidence/Task%201.png)

---

## Task 2: Centralise Logs

Once `auth.log` was created, the log lines were shipped to LocalStack CloudWatch Logs using the AWS CLI.

```bash
TS=$(date +%s000)
while IFS= read -r line; do
  aws $EP logs put-log-events --log-group-name /ccse/app --log-stream-name auth \
    --log-events timestamp=$TS,message="$line" >/dev/null
  TS=$((TS+1000))
done < auth.log
```

The log stream was then read back for verification:

```bash
aws $EP logs get-log-events --log-group-name /ccse/app --log-stream-name auth \
  --query 'events[].message' --output text
```

### Verification Outcome

The command returns the same authentication events from the central log store. This confirms that telemetry collected on the local host has been centralised into a single CloudWatch Logs stream for monitoring, searching, and forensic review.

### Evidence Screenshot

![Ship logs to CloudWatch Logs](Evidence/Task%202.png)

---

## Task 3: Query for Security-Relevant Activity

The next step is to identify the suspicious source IP by counting failed login attempts grouped by address.

```bash
grep LOGIN_FAIL auth.log | awk '{print $4, $5}' | sort | uniq -c
```

This pipeline extracts the failed-login records, prints the relevant fields, sorts them, and counts occurrences per source IP. Example result:

```text
4 ip=203.0.113.9
```

### Security Relevance

The repeated failed attempts from a single IP address pattern is a classic brute-force signal. It often indicates password guessing or credential stuffing. In this lab, the IP is later validated by the presence of a successful login and export event from the same address.

### Evidence Screenshot

![Failed login query](Evidence/Task%203.png)

---

# Session B: Tamper-Proofing, Detection and Response

## Task 4: Tamper-Proof (Hash-Chained) Logs

Audit integrity must be protected because an attacker may manipulate the log file after access. The lab used a SHA-256 hash chain to create a tamper-evident trail.

```bash
PREV=0
while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
  printf '%s | %s\n' "$line" "$PREV"
done < auth.log > auth.chain
cat auth.chain
```

A tampered file was then created by changing the export size from `500MB` to `5MB`:

```bash
sed 's/500MB/5MB/' auth.log > auth.tampered
```

The manipulated content breaks the hash chain because each line depends on the previous hash. If the attacker edits an earlier entry, the final hash differs from the expected value.

### Verification Principle

The objective is to detect tampering automatically by comparing the current chain to the expected chain. Any modification in the log content alters the hash values downstream, making unauthorised change visible.

### Evidence Screenshot

![Hash-chained and tampered logs](Evidence/Task%204.png)

---

## Task 5: Detect the Incident by Correlation

The incident was detected using a correlation rule to identify the pattern:

- `fails >= 3`
- `success >= 1`
- `export >= 1`
- all associated with the same source IP: `203.0.113.9`

```bash
IP=203.0.113.9
FAILS=$(grep -c "LOGIN_FAIL.*$IP" auth.log)
SUCCESS=$(grep -c "LOGIN_OK.*$IP" auth.log)
EXPORT=$(grep -c "EXPORT_DATA.*$IP" auth.log)
echo "IP=$IP fails=$FAILS success=$SUCCESS export=$EXPORT"

if [ "$FAILS" -ge 3 ] && [ "$SUCCESS" -ge 1 ] && [ "$EXPORT" -ge 1 ]; then
  echo 'ALERT: probable brute-force -> compromise -> data exfiltration'
fi
```

### Observed Output

```text
IP=203.0.113.9 fails=4 success=1 export=1
ALERT: probable brute-force -> compromise -> data exfiltration
```

### Interpretation

A single log entry might not prove malicious intent. However, when correlated, the sequence shows a coherent story:

1. the source IP initiated repeated failed attempts,
2. succeeded in logging in,
3. then triggered a large export event.

This is consistent with a comprehensive attack chain and is precisely the kind of alert that a SIEM or security analytics platform should raise.

### Evidence Screenshot

![Incident correlation alert](Evidence/Task%205.png)

---

## Task 6: Incident Response

The response phase includes containment and evidence preservation.

### Containment via Firewall Rule

An Alpine Linux container was used to add a firewall rule to block the attacker’s IP.

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c \
  'apk add -q iptables; iptables -A INPUT -s 203.0.113.9 -j DROP; iptables -L INPUT -n | tail -2'
```

The resulting rule:

```text
DROP  all  --  203.0.113.9  0.0.0.0/0
```

This operation prevents ingress from the suspected malicious IP and limits further attack traffic.

### Forensic Collection and Integrity Protection

A forensic copy was made and hashed to preserve evidence integrity.

```bash
cp auth.log evidence_$(date +%Y%m%d).log
sha256sum evidence_*.log > evidence.sha256
cat evidence.sha256
```

Example output:

```text
0adc5d2ac06cbbdd366099bcc0540c4c0f76946e71b52e4c99322731696a203b  evidence_20260902.log
```

This gives investigators a verifiable evidence artefact that can be checked later using `sha256sum -c evidence.sha256`.

### Evidence Screenshot

![Incident response and evidence hash](Evidence/Task%206.png)

---

# Incident Report

## Detection

The incident was detected by correlating secure authentication telemetry and export activity for the account `admin`. The source IP `203.0.113.9` generated four `LOGIN_FAIL` entries followed by one successful login and a large `EXPORT_DATA` with a size of `500MB`. The threshold for detection was met under the rule: failed logins `>= 3`, successful login `>= 1`, export event `>= 1` from the same IP. This combination generated an `ALERT` and indicated a probable compromise sequence.

## Analysis

The event pattern is consistent with a brute-force attack followed by successful account access and subsequent exfiltration. The threat profile aligns with common attacker behaviour: enumerate credentials, identify a valid account, and then use the compromised access to move sensitive data out of the environment. The `admin` account was the target, and the same remote source repeated the malicious behaviour across the authentication and export stages.

## Containment

Containment was implemented via firewall filtering. The rule `iptables -A INPUT -s 203.0.113.9 -j DROP` prevented further inbound traffic from the suspect IP. This action reduced the probability of additional access attempts and gave the security team a controlled window to preserve forensic evidence and investigate the intrusion path.

## Evidence and Integrity

Collection of forensic evidence included copying the relevant log file to a timestamped evidence file such as `evidence_20260902.log` and computing a SHA-256 digest to `evidence.sha256`. This preserves non-repudiation and demonstrates that the evidence was not modified after collection. The integrity check can later confirm whether any change occurred during analysis or storage.

## Lesson Learned

The lab emphasises that a single failed login or successful login event does not show the full attack story. Centralised logging, audit integrity, and correlation are essential because incidents are often multi-stage and distributed across multiple event sources. Strong account policies such as lockout thresholds and real-time alerting, plus immutable remote SIEM forwarding, would further reduce risk and improve detection coverage.

---

# Verification Commands

The following verification commands were required for the lab:

```bash
aws --endpoint-url=http://localhost:4566 logs describe-log-groups
sha256sum -c evidence.sha256
```

## Expected Interpretations

- `aws --endpoint-url=http://localhost:4566 logs describe-log-groups` should show the log group `/ccse/app` and confirm that telemetry has been centralised.
- `sha256sum -c evidence.sha256` should return an `OK` result for the preserved evidence file if no modification has occurred.

### Evidence Screenshot

![Verification commands](Evidence/Verification%20Commands.png)

---

# Short-Answer Questions

## Q1: What is the difference between a log and an event? Give an example of each from this lab.

A log is a raw, time-stamped record of an occurrence. In the lab, the following line is a log entry:

```text
2025-03-01T09:01:15 LOGIN_FAIL user=admin ip=203.0.113.9
```

This is a single event record generated by the authentication system. An event, by contrast, is a higher-level interpretation or correlation of one or more log entries. For example, the pattern of four failed logins from `203.0.113.9`, followed by a successful login and data export, forms a security event: a likely brute-force compromise and exfiltration. The distinction is that logs are the evidence sources while events are the actionable security conclusions derived by analysis.

## Q2: Why must audit logs be tamper-proof, and how does a hash chain guarantee integrity?

Audit logs must be tamper-proof because they are used as evidence for legal, operational, and incident response purposes. If attackers can edit or delete the log trail, they can conceal malicious activity and invalidate the investigation. A hash chain protects integrity by producing a digest that depends on both the current log entry and the previous hash value. If one line is modified, the hash values after that point are invalidated. The effect is that even a small change becomes visible, which supports non-repudiation and authentic forensic review.

## Q3: How did event correlation detect an incident that no single log line revealed?

No individual log line conclusively proves malicious exfiltration. However, the correlation of multiple entries revealed the sequence:

- repeated `LOGIN_FAIL` events from same IP,
- followed by `LOGIN_OK` from same source,
- followed by `EXPORT_DATA` event.

This pattern could not be fully understood from a single record. The detection logic aggregated related behaviour by the common source IP and raised an alert when all three conditions were met. That is how SIEM tooling transforms discrete log records into a meaningful incident story.

## Q4: List the incident-response steps you performed and the objective of each.

1. Detection: identify suspicious behaviour by querying failed login activity and correlation rules.
2. Analysis: determine the source IP, target account, and probable attack path.
3. Containment: block the attacker with `iptables -A INPUT -s 203.0.113.9 -j DROP`.
4. Evidence collection: copy the relevant log file to a timestamped forensic copy.
5. Integrity protection: compute `sha256sum` values to preserve evidence authenticity.
6. Documentation: record findings, impact, and containment actions for formal reporting.

The objective of this lifecycle is to stop active harm while preserving enough evidence to investigate root cause and support audit or legal accountability.

## Q5: How do shared telemetry sources serve both security monitoring and regulatory compliance?

The same telemetry supports both functions because logs capture operational behaviour and security-relevant events in a standard, time-ordered form. For monitoring, the logs enable detection of failed logins, privilege misuse, and data export events. For compliance, the same logs provide an auditable record showing who accessed what and when. Centralisation, retention, and hash integrity strengthen this dual purpose by ensuring that the evidence is not easily changed or deleted. In cloud environments, this is essential because regulators and internal governance controls expect demonstrable visibility and accountability.

---

# Security Best-Practices Checklist

- [x] Logs were centralised to a managed backend using LocalStack CloudWatch Logs.
- [x] Application telemetry was collected for authentication and export events.
- [x] Security-relevant activity was queried using `grep` and `awk`.
- [x] Failed logins were grouped by source IP to detect brute-force patterns.
- [x] Audit data was protected using SHA-256 hash chaining.
- [x] Log tampering was demonstrated by comparing the original and tampered chains.
- [x] Event correlation was used to identify a multi-stage incident.
- [x] The suspected source IP was contained using `iptables`.
- [x] Evidence was preserved with a timestamped copy and hash record.
- [x] The incident report includes detection, analysis, containment, integrity, and lessons learned.

---

# Cleanup and Teardown Commands

After completion, remove the temporary lab artefacts and stop the LocalStack container.

```bash
rm -f auth.log auth.chain auth.tampered evidence_*.log evidence.sha256

docker stop localstack && docker rm localstack
```

---

# Conclusion

This lab demonstrated the complete monitoring and incident-response workflow for a cloud security scenario. The student generated realistic authentication telemetry, centralised it to LocalStack CloudWatch Logs, queried it for suspicious activity, protected it with SHA-256 hash chaining, and correlated the events to detect a probable brute-force attack followed by data exfiltration. The response phase further showed how containment and forensic evidence collection preserve the integrity of investigation while reducing ongoing attacker access.

The primary lesson is that cloud security depends on more than preventive controls. Visibility, integrity, correlation, and disciplined incident response are necessary to detect, contain, and document malicious activity effectively.
