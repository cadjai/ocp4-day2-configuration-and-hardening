# Day 2 STS Configuration (Configuring AWS STS for Post-Deployment OpenShift Components)

This guide provides a step-by-step walkthrough for granting OpenShift workloads and Operators access to AWS resources (such as Amazon S3, EFS, ...) on a cluster configured with **AWS Security Token Service (STS)** authentication.

---

## 1. Overview & Architecture

When OpenShift runs on AWS in **STS mode**, pods do not use static AWS IAM access keys (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`). Instead, they authenticate via **Web Identity Federation (OIDC)**:

1. The pod's Kubernetes **ServiceAccount (SA)** projects a signed OpenShift service account token into the container.
2. The AWS SDK or CLI exchanges this token with AWS STS for temporary credentials by assuming a dedicated IAM role.
3. The IAM role's **Trust Relationship** restricts access strictly to the component's OpenShift namespace and ServiceAccount.

### The 3-Step Lifecycle

```
┌─────────────────────────────────┐
│ Step 1: Discovery & Scoping     │ Identify AWS resources, required IAM actions,
│                                 │ target namespace, and ServiceAccount(s).
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ Step 2: IAM & Trust Provisioning│ Define `ocp_comp_sts_to_configure` variables,
│                                 │ run Ansible Day-2 playbook (native or ccoctl).
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ Step 3: Workload Configuration  │ Patch Operator Subscription or annotate SAs,
│                                 │ and provision backing S3 secrets/buckets.
└─────────────────────────────────┘
```

---

## 2. Prerequisites

Ensure your bastion or deployment runner has the following prerequisites configured:

- **OpenShift CLI (`oc`)**: Authenticated with cluster administrator permissions.
- **AWS CLI (`aws`)**: Configured with credentials permitted to create and retrieve IAM roles/policies.
- **Ansible Core 2.12+**: Configured with necessary collections (`ansible.builtin`, `community.aws` if applicable).
- **Cluster Information**: Cluster name (`cluster_name`), AWS Region (`aws_region`), and existing OIDC provider URL associated with the cluster.

---

## 3. Step 1: Component Scoping & Requirements Gathering

Before running any playbooks, collect the following technical attributes for the component:

| Attribute | Description | Example |
| :--- | :--- | :--- |
| **Component Name** | Internal identifier for the workload | `loki-storage`, `velero`, `thanos` |
| **Namespace** | Target namespace where the workload or operator resides | `openshift-logging`, `openshift-adp` |
| **Service Account(s)** | Kubernetes ServiceAccounts used by the pods | `loki-operator`, `loki-collector` |
| **IAM Policy / Permissions** | Actions required (e.g., S3 PutObject, EFS Mount) | Custom JSON policy or list of AWS service actions |
| **Integration Method** | Native template vs. Cloud Credential Operator (`ccoctl`) | `use_ccoctl: true` or `use_ccoctl: false` |
| **Operator Subscription** | (If operator-based) Subscription object name | `loki-operator` |

---

## 4. Step 2: Ansible Variable Configuration & IAM Provisioning

Provisioning is automated using the playbook:
`post-deploy-configure-ocp-credentials-for-components-needing-sts.yml`.

### 4.1. Define Component Inventory (`vars/`)

Add your component entry to the `ocp_comp_sts_to_configure` dictionary within your Ansible variable structure in `vars/cluster.yml` .

#### Option A: Using Pre-defined IAM Policy Files (`use_ccoctl: false`)
Use this mode when you have specific custom IAM JSON policies or as default until the ccotl bug related to secured AWS regions is fixed:

```yaml
ocp_comp_sts_to_configure:
  loki:
    name: "loki"
    namespace: "openshift-logging"
    service_accounts:
      - "loki-operator"
      - "loki-store"
    iam_perms_policy_doc_file: "templates/iam-policies/loki-s3-policy.json.j2"
    aws_cloud_cred_secret_name: "logging-loki-s3-credentials"
    aws_service_perms_uselist: "false"
    use_ccoctl: false
```

#### Option B: Using `ccoctl` Automated Generation (`use_ccoctl: true`)
Use this mode when leveraging OpenShift's Cloud Credential Operator tooling in non secured AWS region or when the bug related to this tool in AWS secured regions is fixed:

```yaml
ocp_comp_sts_to_configure:
  adp_velero:
    name: "velero"
    namespace: "openshift-adp"
    service_accounts:
      - "velero"
    aws_service_perms: "s3:*"
    aws_cloud_cred_secret_name: "cloud-credentials"
    use_ccoctl: true
```

### 4.2. (Optional) Define Service Accounts to Annotate

If the component requires explicit role annotations on its Kubernetes ServiceAccount instead of Operator environment variables:

```yaml
ocp_comp_sts_sa_to_annotate:
  loki_sa:
    namespace: "openshift-logging"
    sa_name: "loki-operator"
    iam_role_name: "{{ cluster_name }}-openshift-loki"
    # Or explicitly provide iam_role_arn if known
    # iam_role_arn: "arn:aws:iam::123456789012:role/my-custom-role"
```

### 4.3. Run the Provisioning Playbook

Execute the Day-2 credentials playbook:

```bash
ansible-playbook -i localhost \
  playbooks/post-deploy-configure-ocp-credentials-for-components-needing-sts.yml \
  --ask-vault-pass
```

**What this playbook executes:**
1. Validates dictionary definitions and checks for an active OpenShift session.
2. Creates staging directories under `{{ staging_dir }}/{{ cluster_name }}/`.
3. Ensures target namespaces exist.
4. Generates the AWS IAM Role with a Trust Policy linked to the cluster's OIDC issuer and specified ServiceAccounts.
5. Generates/applies the AWS IAM Secret manifests or `CredentialsRequest` manifests.
6. Annotates specified ServiceAccounts with the IAM Role ARN (if configured).

---

## 5. Step 3: Workload Consumption & Operator Configuration

Once the IAM role exists in AWS, the running component must be instructed to consume it.

### Scenario A: Operator Configuration via Subscription Patching

For Operator Lifecycle Manager (OLM) Operators, inject the STS environment variables directly into the Operator `Subscription.spec.config.env`.

#### What the Subscription Task Does
The task resolves the IAM Role ARN dynamically and patches the subscription:

```yaml
spec:
  config:
    env:
      - name: ROLEARN
        value: "arn:aws:iam::<ACCOUNT_ID>:role/<CLUSTER_NAME>-openshift-<COMPONENT>"
      - name: AWS_STS_REGIONAL_ENDPOINTS
        value: "regional"
      - name: AWS_REGION
        value: "<AWS_REGION>"
      - name: AWS_DEFAULT_REGION
        value: "<AWS_REGION>"
```

#### Ansible task file to patch/configure the subscription
Import the task into the appropriate playbook using a snippet like below:
```
   - name: '{{ ansible_name_module }} | include_tasks | Import subscription STS config tasks'
      ansible.builtin.include_tasks: tasks/configure-subscription-for-sts.yml
      when:
        - use_sts_creds is defined
        - use_sts_creds | bool
      vars:
        comp_name: '<comp-name>'
        comp_sub_name: '<subscription-name>'
        comp_ns: '<comp-namespace>'
        create_s3_and_secret: 'false'
```
Run the updated playbook so the subscription is patched.

#### Manual CLI Equivalent
```bash
ROLE_ARN=$(aws iam get-role --role-name "${CLUSTER_NAME}-openshift-loki" --query 'Role.Arn' --output text)

oc patch subscription.operators.coreos.com loki-operator -n openshift-logging --type='merge' -p "{
  \"spec\": {
    \"config\": {
      \"env\": [
        {\"name\": \"ROLEARN\", \"value\": \"${ROLE_ARN}\"},
        {\"name\": \"AWS_STS_REGIONAL_ENDPOINTS\", \"value\": \"regional\"},
        {\"name\": \"AWS_REGION\", \"value\": \"${AWS_REGION}\"},
        {\"name\": \"AWS_DEFAULT_REGION\", \"value\": \"${AWS_REGION}\"}
      ]
    }
  }
}"
```

---

### Scenario B: Storage Backing (S3 Bucket & Component Secrets)

When components (like Thanos, Loki, ODF, or Velero) require dedicated S3 buckets along with connection parameters:

1. Enable `create_s3_and_secret: true` in your playbook execution variables.
2. Provide the following component inputs:
   - `comp_s3_bucket`: S3 bucket name (e.g., `mycluster-loki-storage`).
   - `comp_s3_secret`: Secret name to be mounted by the workload (e.g., `loki-s3-secret`).
   - `comp_cr_ns`: Namespace where the target Custom Resource (CR) and secret reside.

```
   - name: '{{ ansible_name_module }} | include_tasks | Import subscription STS config tasks'
      ansible.builtin.include_tasks: tasks/configure-subscription-for-sts.yml
      when:
        - use_sts_creds is defined
        - use_sts_creds | bool
      vars:
        comp_name: '<comp-name>'
        comp_sub_name: '<subscription-name>'
        comp_ns: '<comp-namespace>'
        create_s3_and_secret: 'true'
        comp_s3_bucket: '<comp-s3-bucket-name>'
        comp_s3_secret: '<comp-s3-bucket-secret>'
        comp_cr_ns: '<comp-s3-bucket-secret-namespace>'
```

#### Resulting Secret Format
The playbook generates a generic Kubernetes secret containing the STS metadata without static credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: loki-s3-secret
  namespace: openshift-logging
type: Opaque
stringData:
  bucket: "mycluster-loki-storage"
  region: "us-east-1"
  role_arn: "arn:aws:iam::123456789012:role/mycluster-openshift-loki"
```

---

## 6. Verification and Health Checks

Verify your configuration using the following inspection checklist:

### 1. Verify IAM Trust Relationship in AWS
Run the following AWS CLI command to ensure the OpenShift OIDC provider and ServiceAccounts are listed under the Trust Policy:

```bash
aws iam get-role --role-name <ROLE_NAME> --query 'Role.AssumeRolePolicyDocument' --output json
```

Expected snippet:
```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER>"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "<OIDC_PROVIDER>:sub": "system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>"
    }
  }
}
```

### 2. Verify Operator Pod Environment Variables
Check that the operator pod restarted and received the injected environment variables:

```bash
oc get pods -n <NAMESPACE> -l name=<OPERATOR_NAME> -o jsonpath='{.items[0].spec.containers[0].env}' | jq .
```

### 3. Verify Projected Token Mount in Workload Pods
Workload pods assuming the role will automatically mount an OIDC token and have AWS variables set:

```bash
oc get pod <WORKLOAD_POD> -n <NAMESPACE> -o jsonpath='{.spec.containers[*].volumeMounts}' | jq .
```
Verify that:
- `/var/run/secrets/openshift/serviceaccount/token` or standard AWS token projection exists.
- `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` are set if managed by the CCO pod admission webhook.

---

## 7. Troubleshooting

| Issue | Root Cause | Resolution |
| :--- | :--- | :--- |
| **`AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity`** | ServiceAccount name or namespace mismatch in the IAM trust relationship. | Inspect `AssumeRolePolicyDocument`. Ensure the `<OIDC_PROVIDER>:sub` value exactly matches `system:serviceaccount:<namespace>:<serviceaccount>`. |
| **Role Name Length Limit Exceeded (> 64 characters)** | IAM role names cannot exceed 64 characters. | Ensure `cluster_name + '-openshift-' + comp_name` is within 64 chars. The playbook enforces `[:64]` truncation on computed role names. |
| **Subscription patch has no effect on running operator** | OLM operator pod was not restarted after Subscription config change. | Delete the operator deployment pod to force recreation: `oc delete pod -l name=<operator-name> -n <namespace>`. |
| **`InvalidIdentityToken`** | The cluster's OIDC issuer URL in AWS IAM Identity Provider is incorrect or expired. | Verify `oc get authentication cluster -o jsonpath='{.spec.serviceAccountIssuer}'` against the IAM Identity Provider in the AWS Console. |


### Scenario C: Component is not managed by an operator and might not use an AWS secret

When components are not managed by an operator or an OpenShift controller you will need to identify the appropriate deployment type (e.g. deployment, statefulset, daemonset...) and then update the required environment variables to ensure that it is able to use the created IAM role as well as potentially update the service account to have the correct annotations.  
Below are some example environment variables that might need to be configured
```
  "ROLEARN": "${ROLE_ARN}"
  "AWS_STS_REGIONAL_ENDPOINTS": "regional"
  "AWS_REGION": "${AWS_REGION}"
  "AWS_DEFAULT_REGION": "${AWS_REGION}"
```
In addition there might be a need to update annotations on the service account and potentially create any necessary secrets. 
