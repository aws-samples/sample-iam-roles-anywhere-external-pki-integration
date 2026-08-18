# IAM Roles Anywhere — External PKI Integration

This sample deploys a complete [AWS IAM Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html) setup for workloads that authenticate using X.509 certificates issued by an external Certificate Authority (CA). It implements the security best practices described in the AWS Security Blog post: *"Best practices for integrating IAM Roles Anywhere with your PKI"*.

## Architecture

The template creates:

1. **Trust Anchor** — Anchors trust to your external CA certificate (CERTIFICATE_BUNDLE mode).
2. **IAM Role** — Assumable only by workloads presenting certificates with matching identity attributes (Issuer CN, Subject CN, SAN CN) and scoped to the specific trust anchor.
3. **Roles Anywhere Profile** — Links the role with an inline session policy that enforces encrypted transport and region restriction.
4. **CloudWatch Alarm** — Monitors `CreateSession` failures (potential certificate issues or unauthorized attempts).
5. **EventBridge + SNS (optional)** — Alerts on trust anchor certificate approaching expiry.

### Architecture Diagram

```
┌─────────────────────┐         ┌──────────────────────────────┐
│  External Workload  │         │      AWS Account             │
│                     │         │                              │
│  ┌───────────────┐  │         │  ┌────────────────────────┐  │
│  │ X.509 Cert    │──┼────────►│  │  Trust Anchor          │  │
│  │ (CA-signed)   │  │         │  │  (Your CA cert)        │  │
│  └───────────────┘  │         │  └───────────┬────────────┘  │
│                     │         │              │               │
│  ┌───────────────┐  │         │  ┌───────────▼────────────┐  │
│  │ Private Key   │  │         │  │  IAM Role              │  │
│  │ (PKCS#11/TPM) │  │         │  │  (Certificate-scoped)  │  │
│  └───────────────┘  │         │  └───────────┬────────────┘  │
│                     │         │              │               │
│  aws_signing_helper │         │  ┌───────────▼────────────┐  │
│  credential-process │         │  │  Profile               │  │
│                     │         │  │  (Session policy)      │  │
└─────────────────────┘         │  └────────────────────────┘  │
                                │                              │
                                │  ┌────────────────────────┐  │
                                │  │  CloudWatch Alarm      │  │
                                │  │  (CreateSession fails) │  │
                                │  └────────────────────────┘  │
                                └──────────────────────────────┘
```

## Security Controls Implemented

| Control | Implementation |
|---------|---------------|
| Certificate-based access control | Trust policy with `x509Issuer/CN`, `x509Subject/CN`, `x509SAN/Name/CN` conditions |
| Trust anchor scoping | `ArnEquals` condition restricts to the specific trust anchor |
| Session boundary | Inline session policy denies unencrypted transport and restricts to deployment region |
| Least privilege | Role grants only S3 read + STS:GetCallerIdentity |
| Monitoring | CloudWatch Alarm on CreateSession failures |
| Certificate lifecycle | EventBridge rule for trust anchor cert expiry notifications |

## Prerequisites

- An AWS account with permissions to deploy CloudFormation stacks with IAM resources.
- A PEM-encoded X.509 CA certificate from your external PKI.
- A workload certificate signed by that CA with the following attributes:
  - Subject CN (e.g., `WorkloadA`)
  - Issuer CN matching your CA (e.g., `MyCompany CA`)
  - Subject Alternative Name with a CN
- [AWS IAM Roles Anywhere credential helper](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/credential-helper.html) installed on the workload host.

## Deployment

### Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `TrustAnchorCertificateBody` | Yes | — | PEM-encoded CA certificate (including BEGIN/END markers) |
| `WorkloadSubjectCN` | No | `WorkloadA` | Subject CN from the workload certificate |
| `WorkloadIssuerCN` | No | `MyCompany CA` | Issuer CN from the CA certificate |
| `WorkloadSANCN` | No | `WorkloadA` | SAN CN from the workload certificate |
| `SessionDurationSeconds` | No | `3600` | Maximum session duration (900–43200) |
| `NotificationEmail` | No | *(empty)* | Email for certificate expiry alerts |

### Deploy via AWS CLI

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name iamra-pki-integration \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    TrustAnchorCertificateBody="$(cat ca-certificate.pem)" \
    WorkloadSubjectCN="WorkloadA" \
    WorkloadIssuerCN="MyCompany CA" \
    WorkloadSANCN="WorkloadA" \
    NotificationEmail="security-team@example.com"
```

### Deploy via AWS Console

1. Navigate to **CloudFormation** → **Create stack** → **With new resources**.
2. Upload `template.yaml`.
3. Enter your CA certificate PEM and workload identity attributes.
4. Acknowledge IAM capabilities and create the stack.

## Testing the Deployment

After deployment, configure the credential helper on your workload:

```ini
# ~/.aws/config
[profile roles-anywhere]
credential_process = aws_signing_helper credential-process \
  --certificate /path/to/workload-cert.pem \
  --private-key /path/to/workload-key.pem \
  --trust-anchor-arn <TrustAnchorArn from stack outputs> \
  --profile-arn <ProfileArn from stack outputs> \
  --role-arn <WorkloadRoleArn from stack outputs>
```

Verify it works:

```bash
aws sts get-caller-identity --profile roles-anywhere
```

Expected output shows the assumed role with the workload identity:

```json
{
  "UserId": "AROA...:WorkloadA",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/iamra-pki-integration-workload-role/WorkloadA"
}
```

## Cleanup

```bash
aws cloudformation delete-stack --stack-name iamra-pki-integration
```

## Related Resources

- [Best practices for integrating IAM Roles Anywhere with your PKI](https://aws.amazon.com/blogs/security/) *(companion blog post)*
- [Planning for your IAM Roles Anywhere deployment](https://aws.amazon.com/blogs/security/planning-for-your-iam-roles-anywhere-deployment/) *(April 2025)*
- [IAM Roles Anywhere documentation](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html)
- [Credential helper installation guide](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/credential-helper.html)

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
