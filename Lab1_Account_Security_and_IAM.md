# Lab 1: Cloud Account Security, Identity and Access Management

**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab:** Lab 1
**Topic:** Identity governance, least privilege, LocalStack IAM and Kubernetes RBAC  
**Environment:** LocalStack on `localhost:4566` and kind Kubernetes cluster `ccse-lab1`  
**Name:** MUHAMMAD DANISH ISYRAQ BIN AB GHANI

## Lab Summary // Objective

This lab validates core cloud account security practices through hands-on work with local IAM and RBAC systems. The objectives are to:

- Map cloud identity constructs to real IAM concepts.
- Create least-privilege administrative access using LocalStack IAM.
- Create and scope a read-only user and manage access keys securely.
- Deploy Kubernetes namespaces, roles, and role bindings to enforce RBAC.
- Verify authorization decisions and document the access control configuration.

## Evidence Summary Table

| Evidence File | Purpose |
|---|---|
| `2.1.png` | Confirm creation of the `Admins` IAM group in LocalStack. |
| `2.2.png` | Confirm creation of the `CloudAdmin_dani` IAM user. |
| `2.3.png` | Confirm attaching administrator policy to `Admins` and group binding actions. |
| `2.4.png` | Confirm `CloudAdmin_dani` membership in the `Admins` group. |
| `3.1.png` | Confirm creation of the `Analyst_jiha` IAM user. |
| `3.2.png` | Confirm attaching `AmazonS3ReadOnlyAccess` policy to `Analyst_jiha`. |
| `3.3.png` | Confirm the permissions attached to `Analyst_jiha`. |
| `4.1.png` | Confirm creation of an access key for `Analyst_jiha`. |
| `4.2.png` | Confirm access key listing for `Analyst_jiha`. |
| `4.3.png` | Confirm deactivation of the old access key for credential rotation. |
| `5.png` | Confirm Local Kubernetes cluster creation and cluster info. |
| `5.0.png` | Confirm creation of the `dev` and `prod` namespaces. |
| `5.1.png` | Confirm namespace listing and status. |
| `6.1.png` | Confirm creation of the `dev-user` service account. |
| `6.2.png` | Confirm creation of the `pod-reader` Role in namespace `dev`. |
| `6.3.png` | Confirm creation of the `dev-user-binding` RoleBinding. |
| `7.png` | Confirm RBAC authorization test results for the service account. |
| `Verification-RBAC.png` | Confirm YAML output for the `dev-user-binding` RoleBinding. |

## Task 1: Map the Cloud Identity Landscape

| Concept | AWS Term | Purpose |
|---|---|---|
| Account owner with full control | Root user | The root owner has unrestricted access to all resources and billing; it must be protected and reserved for emergency use only. |
| Named human or application identity | IAM User | A permanent identity for a person or application that can authenticate with long-lived credentials. |
| Permission definition | IAM Policy | A JSON document that grants or denies access to cloud actions, resources, and conditions. |
| Group of users | IAM Group | A collection of users that can share a common set of attached policies for easier privilege management. |
| Delegated temporary access | IAM Role | A credential-less identity that can be assumed to grant short-lived permissions to users, services, or applications. |

## Session A: LocalStack IAM

### Environment Setup

The LocalStack endpoint was configured for AWS CLI with the environment variable:

```bash
EP='--endpoint-url=http://localhost:4566'
```

This routes AWS CLI commands to LocalStack instead of AWS. The following command verifies the LocalStack IAM environment:

```bash
aws $EP sts get-caller-identity
```

Example output:

```json
{
    "UserId": "000000000000",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

The account ID `000000000000` confirms the local endpoint.

## Task 2: Create a Least-Privilege Admin

### Step 2.1: Create Admins Group

Command:

```bash
aws $EP iam create-group --group-name Admins
```

Result:

The `Admins` group was created successfully in LocalStack.

Evidence:

<img width="515" height="267" alt="2 1" src="https://github.com/user-attachments/assets/302e3ac9-62a8-4776-bf7d-903e9cd84582" />

### Step 2.2: Attach Administrator Policy to Group

Command:

```bash
aws $EP iam attach-group-policy --group-name Admins \
    --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

Verification command:

```bash
aws $EP iam list-attached-group-policies --group-name Admins
```

Expected output:

```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AdministratorAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
        }
    ]
}
```

This confirms the `Admins` group has administrator privileges attached.

Evidence:

<img width="645" height="207" alt="2 2" src="https://github.com/user-attachments/assets/7863651d-46dd-4866-b8a2-c7b9e555f5ae" />

### Step 2.3: Create Personal Admin User

Command:

```bash
aws $EP iam create-user --user-name CloudAdmin_dani
```

Result:

The IAM user `CloudAdmin_dani` was created successfully.

Evidence:

<img width="556" height="67" alt="2 3" src="https://github.com/user-attachments/assets/af245875-911f-457e-8143-d5d18bab3391" />

### Step 2.4: Add User to Admins Group and Verify Membership

Command:

```bash
aws $EP iam add-user-to-group --group-name Admins \
    --user-name CloudAdmin_dani
```

Verification command:

```bash
aws $EP iam get-group --group-name Admins
```

Expected output summary:

```json
{
    "Users": [
        {
            "UserName": "CloudAdmin_dani",
            "Arn": "arn:aws:iam::000000000000:user/CloudAdmin_dani"
        }
    ],
    "Group": {
        "GroupName": "Admins",
        "Arn": "arn:aws:iam::000000000000:group/Admins"
    }
}
```

This confirms `CloudAdmin_dani` is a member of the `Admins` group. The user inherits administrator privileges from the group rather than receiving direct admin permissions.

Evidence:

<img width="646" height="348" alt="2 4" src="https://github.com/user-attachments/assets/2e921c03-224c-488d-89f1-7b12a9fce4d5" />

## Task 3: Enforce Least Privilege with a Scoped Policy

### Step 3.1: Create Read-Only Analyst User

Command:

```bash
aws $EP iam create-user --user-name Analyst_jiha
```

Result:

The IAM user `Analyst_jiha` was created successfully.

Evidence:

<img width="597" height="206" alt="3 1" src="https://github.com/user-attachments/assets/0f08c687-e975-4893-a7c5-0926d2a4a21a" />

### Step 3.2: Attach S3 Read-Only Policy

Command:

```bash
aws $EP iam attach-user-policy --user-name Analyst_jiha \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### Step 3.3: Verify Analyst Permissions

Verification command:

```bash
aws $EP iam list-attached-user-policies --user-name Analyst_jiha
```

Expected output:

```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonS3ReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
        }
    ]
}
```

This proves `Analyst_jiha` only has the scoped read-only S3 permissions.

Evidence:

<img width="622" height="200" alt="3 3" src="https://github.com/user-attachments/assets/0974e7c7-dd8b-4817-93e8-fc2a396546e6" />

### Least Privilege Explanation

- The `Analyst_jiha` account is limited to read-only S3 operations, reducing the damage if the account is compromised.
- Without administrative or write privileges, the account cannot modify IAM settings, create resources, or delete data.
- This scoped access helps enforce least privilege and limits the blast radius of credential misuse.

## Task 4: Credential Hygiene and Access Keys

### Step 4.1: Create Access Key

Command:

```bash
aws $EP iam create-access-key --user-name Analyst_jiha
```

Result:

An access key was generated for `Analyst_jiha`.

Evidence:

<img width="636" height="210" alt="4 1" src="https://github.com/user-attachments/assets/d697a7f7-885b-46f6-8940-d5262de9c63b" />

> Security note: The secret access key values are not reproduced in this report. In practice, access keys must never be committed to source control or shared insecurely.

### Step 4.2: List Access Keys

Command:

```bash
aws $EP iam list-access-keys --user-name Analyst_jiha
```

Expected output:

```json
{
    "AccessKeyMetadata": [
        {
            "UserName": "Analyst_jiha",
            "AccessKeyId": "LKIAQAAAAAAANMJV6XA3",
            "Status": "Inactive",
            "CreateDate": "2026-07-29T05:29:06.789002+00:00"
        }
    ]
}
```

Evidence:

<img width="562" height="238" alt="4 2" src="https://github.com/user-attachments/assets/b2a1127f-2330-40ad-bee5-dd7e914c34ce" />

### Step 4.3: Rotate and Deactivate Old Key

Command:

```bash
aws $EP iam update-access-key --user-name Analyst_jiha \
    --access-key-id LKIAQAAAAAAANMJV6XA3 --status Inactive
```

Result:

The access key status was updated to `Inactive`, demonstrating credential rotation and deactivation.

Evidence:

<img width="598" height="137" alt="4 3" src="https://github.com/user-attachments/assets/f3c8a281-ad1e-453d-9523-beb5eb1765fb" />

## Session B: Kubernetes RBAC

### Setup: Create Local Kubernetes Cluster

Commands:

```bash
kind create cluster --name ccse-lab1
kubectl cluster-info --context kind-ccse-lab1
kubectl get nodes
```

Result:

A local `kind` cluster named `ccse-lab1` was created and kubectl context `kind-ccse-lab1` was available.

Evidence:

<img width="683" height="237" alt="5 0" src="https://github.com/user-attachments/assets/3822898b-ef1c-428e-b63e-b098c78c8537" />

## Task 5: Separate Environments with Namespaces

Commands:

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```

Result:

The namespaces `dev` and `prod` were created successfully and reported as `Active`.

Evidence:

<img width="507" height="313" alt="5 1" src="https://github.com/user-attachments/assets/c5f1f04e-21a2-4d0f-9c4b-c2b302faaece" />
<img width="677" height="340" alt="5" src="https://github.com/user-attachments/assets/4a991f4c-16de-4cb9-a895-dd3997227a2b" />

## Task 6: Define a Role and Bind It

### Step 6.1: Create Service Account

Command:

```bash
kubectl create serviceaccount dev-user -n dev
```

Result:

The service account `dev-user` was created in the `dev` namespace.

Evidence:

<img width="462" height="62" alt="6 1" src="https://github.com/user-attachments/assets/73859590-01a3-41b9-9d3a-4716e3353df8" />

### Step 6.2: Create Pod Reader Role

Command:

```bash
kubectl create role pod-reader -n dev \
    --verb=get,list,watch --resource=pods
```

Result:

The `pod-reader` Role was created in namespace `dev` with permissions to `get`, `list`, and `watch` pods.

Evidence:

<img width="500" height="96" alt="6 2" src="https://github.com/user-attachments/assets/c82f29ee-96bd-44c6-a7d9-307364f07645" />

### Step 6.3: Create RoleBinding

Command:

```bash
kubectl create rolebinding dev-user-binding -n dev \
    --role=pod-reader --serviceaccount=dev:dev-user
```

Result:

The RoleBinding `dev-user-binding` was created in the `dev` namespace, binding the `pod-reader` role to the `dev-user` service account.

Evidence:

<img width="570" height="87" alt="6 3" src="https://github.com/user-attachments/assets/5f3d2d53-7e44-42c8-b7f2-47b07c9a431d" />

## Task 7: Test Access Control

The service account identity was expressed as:

```bash
SA=system:serviceaccount:dev:dev-user
```

### Test 7.1: Verify Pod Read Access in Dev

Command:

```bash
kubectl auth can-i get pods --as="$SA" -n dev
```

Expected output:

```text
yes
```

### Test 7.2: Verify Denied Access in Prod

Command:

```bash
kubectl auth can-i get pods --as="$SA" -n prod
```

Expected output:

```text
no
```

These results validate that the `dev-user` service account has access only in the `dev` namespace and not in `prod`.

Evidence:

<img width="488" height="292" alt="7" src="https://github.com/user-attachments/assets/75e57be8-8e3b-46d5-8a12-f5b1e9628490" />

## RBAC Verification YAML

Command:

```bash
kubectl get rolebinding dev-user-binding -n dev -o yaml
```

Expected YAML output:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

Evidence:

<img width="527" height="367" alt="Verification-RBAC" src="https://github.com/user-attachments/assets/4646c366-b541-4289-bae7-74db59135965" />

## Short-Answer Questions

### Q1: What is the difference between identity and access in cloud security?

Identity is the authenticated principal, such as an IAM user, role, or service account. Access is the authorization decision that determines what actions that identity may perform on resources. Identity answers "who" or "what" is requesting access, while access defines "what" that identity is allowed to do.

### Q2: Why is least privilege important and how does it apply to group-based access?

Least privilege reduces the attack surface by granting identities only the permissions needed to perform their job. In group-based access, permissions are attached to groups and users are added to those groups, which centralizes policy management and avoids giving individual users overly broad rights.

### Q3: How do IAM policies and Kubernetes role bindings differ in access control?

IAM policies define explicit permission rules for cloud actions and resources, and they are attached to users, groups, or roles. Kubernetes RoleBindings associate a Role with a subject (user, group, or service account) inside a namespace, enabling RBAC enforcement for cluster resources. IAM policies are cloud-native permission documents; RoleBindings are authorization wiring between identities and RBAC roles.

### Q4: Why should access keys be rotated and managed carefully?

Access keys are long-lived credentials that grant API-level access. Rotating them regularly and deactivating old keys limits exposure if a key is leaked or compromised, and it enforces credential hygiene by ensuring secrets do not remain valid indefinitely.

### Q5: How do namespaces and RBAC improve Kubernetes security?

Namespaces separate workloads and resources into logical trust boundaries, isolating environments such as `dev` and `prod`. RBAC enforces fine-grained permissions within those boundaries, ensuring that identities only access resources in the namespaces where they are authorized.

## Security Best-Practices Checklist

- [x] Protect the root account and avoid using it for daily operations.
- [x] Use IAM groups to assign policies instead of attaching policies directly to users.
- [x] Apply least privilege by limiting roles and policies to only required actions.
- [x] Keep access keys secret, rotate them regularly, and deactivate unused keys.
- [x] Use Kubernetes namespaces to separate environments and reduce blast radius.
- [x] Use service accounts and RoleBindings to restrict access to specific resources.
- [x] Verify permissions using `aws iam list-attached-user-policies` and `kubectl auth can-i`.
- [x] Document authorization configuration and verify binding YAML for auditability.

## Conclusion

This lab demonstrated the application of cloud account security principles through LocalStack IAM and Kubernetes RBAC. The exercise showed how to create and manage least-privilege identities, attach scoped policies, handle access keys responsibly, and enforce namespace-based authorization in Kubernetes. The final RBAC verification confirms that the `dev-user` service account has controlled access to the `dev` namespace only, illustrating the importance of identity governance and access management in secure cloud operations.
