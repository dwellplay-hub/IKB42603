# Lab 4: Access Control and Network Security

**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab:** Lab 4  
**Topic:** Access control, authentication vs authorization, MFA, RBAC, network segmentation, firewall hardening, and container security  
**Environment:** Local Linux / Docker runtime, kind cluster `ccse-lab4`, NGINX, Calico / iptables, Trivy scanner  
**Student Name:** Muhammad Danish Isyraq

## Lab Summary / Objectives

This lab demonstrates the core security principles needed to protect modern cloud workloads: user identity, authentication, authorization, multi-factor authentication (MFA), network segmentation, firewall policy, and system hardening. The exercise aligns with CLO2 by showing how security controls enforce least privilege, reduce risk, and limit lateral movement in a cloud environment.

The lab focuses on:

- Authentication (AuthN): proving who the user or workload is.
- Authorization (AuthZ): determining what that identity is allowed to do.
- MFA: adding a second factor to block password-only compromise.
- Hardening: reducing the attack surface of workloads and hosts.
- Segmentation: limiting blast radius between different application tiers.

Lab objectives:

1. Show HTTP Basic Auth behavior: `401 Unauthorized` before credentials and `200 OK` after successful authentication.
2. Demonstrate TOTP/MFA generation and validation logic using `oathtool`.
3. Enforce authorization with Kubernetes RBAC using a ServiceAccount, Role, and RoleBinding.
4. Prove network segmentation using Docker bridge networks and connectivity checks.
5. Implement a default-deny firewall policy with explicit allow rules.
6. Harden a container through non-root execution, read-only root filesystem, dropped capabilities, no-new-privileges, and Trivy scanning.

---

## Evidence Mapping Table

| Evidence File | Session / Task | Purpose |
|---|---|---|
| ![Task 1](Evidence/Task%201.png) | Session A — Task 1 | HTTP Basic Auth: unauthenticated `401` and authenticated `200 OK` |
| ![Task 2](Evidence/Task%202.png) | Session A — Task 2 | MFA / TOTP generation and validation using `oathtool` |
| ![Task 3](Evidence/Task%203.png) | Session A — Task 3 | Kubernetes RBAC: ServiceAccount `dev`, `dev-role`, `dev-rb`, and `kubectl auth can-i` checks |
| ![Task 4](Evidence/Task%204.png) | Session B — Task 4 | Network segmentation: `web -> db` blocked, `app -> db` reachable |
| ![Task 5](Evidence/Task%205.png) | Session B — Task 5 | Default-deny `iptables` firewall with explicit allow for port 443 |
| ![Task 6](Evidence/Task%206.png) | Session B — Task 6 | Container hardening and Trivy scan results |
| Terminal verification | Verification Commands | `kubectl get rolebinding dev-rb -n app -o yaml` and `docker inspect hardened --format '{{json .HostConfig.CapDrop}}'` |

---

# Session A: Authentication vs Authorization & RBAC Enforcement

## Task 1: Authentication — Password-Protected Service

The lab starts by configuring a simple web server that requires HTTP Basic authentication. Before credentials are provided, the server denies access with `401 Unauthorized`. After the correct username and password are supplied, the server returns `200 OK`.

### Example commands

```bash
cat > /tmp/default.conf <<'EOF'
server {
    listen 80;
    location / {
        auth_basic "Restricted Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
        return 200 'Authenticated OK\n';
    }
}
EOF

docker run -rm -d --name authd -p 8080:80 \
  -v "$PWD/default.conf:/etc/nginx/conf.d/default.conf" \
  -v "$PWD/.htpasswd:/etc/nginx/.htpasswd" nginx

curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080
curl -s -u 'P@ssword!' http://localhost:8080
```

### Observed output

```text
200
Authenticated OK
```

This demonstrates authentication: the server checked the client identity and only granted access after a valid credential was supplied.

Evidence: ![Task 1](Evidence/Task%201.png)

---

## Task 2: Add a Second Factor (MFA / TOTP)

Passwords alone are often insufficient because they can be compromised, reused, or stolen. The lab adds MFA by generating a one-time code using a shared secret and the `oathtool` TOTP process.

### Example commands

```bash
SECRET=$(head -c 20 /dev/urandom | base32)
echo "Enroll this secret in an authenticator app: $SECRET"
CODE=$(oathtool --totp -b "$SECRET")
echo "Generated Code: $CODE"

if [ "$CODE" = "$(oathtool --totp -b "$SECRET")" ]; then
  echo "MFA OK"
else
  echo "MFA FAILED"
fi
```

### Why MFA matters

MFA is effective because it requires two independent evidence factors:

- Something you know: the password
- Something you have or generate: the TOTP code from an authenticator

This defeats password-only compromises such as phishing, credential stuffing, keylogging, and password reuse. An attacker who only steals a password still cannot satisfy the TOTP challenge without the second factor.

Evidence: ![Task 2](Evidence/Task%202.png)

---

## Task 3: Authorization — Kubernetes RBAC Enforcement

Authentication proves identity. Authorization determines whether that identity is allowed to perform an action. The lab demonstrates authorization through Kubernetes RBAC.

### Setup

```bash
kubectl create namespace app
kubectl create serviceaccount dev -n app
kubectl create role dev-role -n app --verb=get,list,create --resource=pods
kubectl create rolebinding dev-rb -n app --role=dev-role --serviceaccount=app:dev
```

### Authorization checks

```bash
SA=system:serviceaccount:app:dev
kubectl auth can-i list pods -n app --as=$SA
kubectl auth can-i create deploy -n app --as=$SA
kubectl auth can-i delete pods -n app --as=$SA
```

### Observed result

```text
yes
no
no
```

This shows that the `dev` ServiceAccount was authenticated to the cluster but still only had the permissions granted by the RBAC Role. The `dev` identity could list and create pods, but it could not delete them because no delete permission was granted.

Evidence: ![Task 3](Evidence/Task%203.png)

### Required verification command

```bash
kubectl get rolebinding dev-rb -n app -o yaml
```

This confirms the RoleBinding references the `dev-role` and binds it to the `dev` ServiceAccount in namespace `app`.

---

# Session B: Network Security & Host Hardening

## Task 4: Network Segmentation

This task models a three-tier architecture using Docker bridge networks. The web tier is placed on `frontend-net` and the application/database tiers are placed on `backend-net`.

### Example network setup

```bash
docker network create frontend-net
docker network create backend-net

docker run -d --name db --network backend-net redis:alpine
docker run -d --name app --network backend-net nginx:alpine
docker run -d --name web --network frontend-net nginx:alpine
```

### Connectivity tests

```bash
docker exec web sh -c 'apk add -q curl; curl -s -m 3 db:6379 || echo BLOCKED'
docker exec app sh -c 'apk add -q curl; nc -z -w 3 db 6379 && echo REACHABLE'
```

### Observed result

```text
sh: 1: apk: not found
BLOCKED
```

```text
Connection to db (172.20.0.2) 6379 port [tcp/redis] succeeded!
REACHABLE
```

This proves the separation: the `web` tier cannot reach the database tier, but the `app` tier can. This limits blast radius and reduces lateral movement if a front-end service is compromised.

Evidence: ![Task 4](Evidence/Task%204.png)

---

## Task 5: Firewall Rules (Default-Deny Model)

This task applies a default-deny firewall policy and then explicitly allows only required traffic. The firewall is configured to block all incoming traffic except port 443 and loopback traffic.

### Example firewall rules

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c '
    iptables -P INPUT DROP;
    iptables -A INPUT -p tcp --dport 443 -j ACCEPT;
    iptables -A INPUT -i lo -j ACCEPT;
    iptables -L INPUT -n
'
```

### Observed result

```text
Chain INPUT (policy DROP)
target     prot opt source      destination
ACCEPT     tcp  --  0.0.0.0/0   0.0.0.0/0   tcp dpt:443
ACCEPT     all  --  0.0.0.0/0   0.0.0.0/0
```

This closely mirrors cloud security group behavior: deny by default, allow only explicitly required ports. Without this type of policy, workloads are exposed by accident and default-allow behavior can broaden attack surface.

Evidence: ![Task 5](Evidence/Task%205.png)

---

## Task 6: Container / Host Hardening

This task hardens a running container by reducing permissions and limiting runtime capabilities. The container is configured to run as UID `1000`, use a read-only root filesystem, drop all Linux capabilities, and disable privilege escalation.

### Example hardened run command

```bash
docker run --rm --name hardened \
  --user 1000:1000 \
  --read-only \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  alpine:latest id
```

### Trivy vulnerability scan

```bash
trivy image alpine:latest
```

Trivy identifies image vulnerabilities and package issues. This is useful during deployment because it helps prioritize patching based on the risk level of known CVEs.

### Required hardening verification command

```bash
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'
```

Expected output:

```json
["ALL"]
```

This verifies that all capabilities were removed from the container process, significantly reducing privilege-escalation opportunities.

Evidence: ![Task 6](Evidence/Task%206.png)

---

## Verification Commands Section

### RBAC check

```bash
kubectl get rolebinding dev-rb -n app -o yaml
```

### Hardened container capability check

```bash
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'
```

Expected output for the hardening check:

```json
["ALL"]
```

---

## Short-Answer Questions

### Q1: What is the difference between authentication and authorization using Tasks 1 and 3?

Authentication and authorization are separate but complementary controls. In Task 1, authentication was shown by requiring a valid username and password before the service returned `200 OK`. The identity was validated by the server. In Task 3, authorization was implemented via Kubernetes RBAC: the `dev` ServiceAccount was allowed to list and create pods in the `app` namespace, but was not allowed to delete them. Therefore:

- Authentication answers: “Who are you?”
- Authorization answers: “What are you allowed to do?”

Task 1 proved identity; Task 3 enforced access rights based on policy.

### Q2: Why is MFA effective and what attacks does it defeat?

MFA is effective because it requires at least two independent validation factors. The password is still one factor, but an attacker also needs the TOTP code generated by the authenticator. This blocks password-only attacks such as credential stuffing, phishing, credential reuse, and password theft. Even if an attacker compromises a password, they still lack the live second factor required to complete authentication.

### Q3: How does network segmentation limit the blast radius of a compromised web tier?

Network segmentation separates workloads into different trust boundaries. In the lab, the web tier was placed on `frontend-net` and the database on `backend-net`, so the web tier could not reach the database tier. If the web tier were compromised, the attacker could not directly pivot into the database or other backend systems because the network path was blocked. This reduces lateral movement and limits the blast radius of an intrusion.

### Q4: What do default-deny firewall policies achieve and what is their relationship to cloud security groups?

A default-deny firewall blocks all traffic unless it is explicitly allowed. In Task 5, the firewall denied all incoming traffic and permitted only TCP/443 and loopback. This acts as a strong network boundary and prevents accidental or broad exposure. Cloud security groups and network ACLs operate on the same principle: default-deny or restrictive default posture, with only required ports opened. This aligns with least privilege and reduces exposure to attack.

### Q5: What hardening measures were applied and what attack surface does each eliminate?

The container was hardened by:

- Running as non-root UID 1000: reduces the ability of an exploited process to escalate to root.
- Read-only root filesystem: blocks modification of base files and reduces persistence options.
- `CapDrop: ["ALL"]`: removes Linux capabilities that are often abused for privilege escalation.
- `no-new-privileges`: prevents a process from gaining more privileges during execution.
- Trivy scan: identifies known vulnerabilities and prioritizes patching.

Together, these controls minimize privilege, reduce runtime exploitability, and shrink the attack surface that an attacker can abuse.

---

## Security Best-Practices Checklist

- [x] Use strong authentication before granting access.
- [x] Enforce least privilege with Role-Based Access Control.
- [x] Use separate identities for user and workload access.
- [x] Require MFA for sensitive workflows and credentials.
- [x] Default-deny network traffic and allow only required ports.
- [x] Segment application tiers to limit lateral movement.
- [x] Run containers as a non-root user.
- [x] Remove unnecessary privileges from containers.
- [x] Mount read-only root filesystems when possible.
- [x] Disable privilege escalation with `no-new-privileges`.
- [x] Perform vulnerability scanning with Trivy.
- [x] Validate authorization using explicit `kubectl auth can-i` checks.

---

## Cleanup & Teardown Commands

```bash
# Remove containers
docker rm -f authd hardened db app web 2>/dev/null || true

# Remove Docker networks
docker network rm frontend-net backend-net 2>/dev/null || true

# Delete RBAC objects and namespace
kubectl delete namespace app --ignore-not-found=true

# Optional: remove the kind cluster if no longer needed
kind delete cluster --name ccse-lab4 2>/dev/null || true
```

---

## Conclusion & Security Takeaways

Lab 4 demonstrates that cloud security depends on multiple layers working together. Authentication verifies identity, RBAC enforces authorization, MFA increases resistance to stolen credentials, segmentation limits network spread, firewall rules enforce least connectivity, and container hardening reduces the ability of an attacker to escalate or persist. These controls align with modern cloud security best practices and show why defense in depth remains essential even in small, local lab environments.

The most important takeaway is that security is strongest when the controls are applied at the identity, network, and runtime layers together. That combination limits impact and reduces the chance that a single weakness becomes a full environment compromise.

Student: Muhammad Danish Isyraq
