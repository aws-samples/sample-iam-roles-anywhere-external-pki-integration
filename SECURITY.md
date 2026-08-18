# Security Policy

## Reporting a Vulnerability

If you discover a potential security issue in this project, we ask that you notify AWS/Amazon Security via our [vulnerability reporting page](http://aws.amazon.com/security/vulnerability-reporting/).

Please do **not** create a public GitHub issue for security vulnerabilities.

## Security Best Practices

When using this sample:

1. **Never commit certificates or private keys** — The `.gitignore` excludes `.pem`, `.key`, `.p12`, and `.pfx` files. Verify before pushing.
2. **Use short-lived certificates** — Prefer certificates with validity under 7 days where possible (AWS Private CA short-lived mode).
3. **Protect private keys** — Store workload private keys in hardware (TPM 2.0, HSM via PKCS#11) rather than on the filesystem.
4. **Rotate trust anchor certificates** — Use Blue/Green trust anchor rotation before CA certificates expire.
5. **Monitor CreateSession failures** — The included CloudWatch Alarm detects potential unauthorized access attempts.
6. **Scope trust policies** — Always include `ArnEquals` on `aws:SourceArn` to prevent other trust anchors from assuming your role.
7. **Apply session policies** — The profile's inline session policy restricts sessions beyond the role's identity policies.

## Supported Versions

Only the latest version of this sample is supported. Please ensure you are using the most recent release before reporting issues.
