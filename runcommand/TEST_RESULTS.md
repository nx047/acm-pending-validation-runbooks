# ACM DNS Validation Checker Test Results

This document records the current manual test results for the Run Command version of the ACM DNS Validation Checker.

Sensitive identifiers such as AWS account ID and certificate UUIDs are intentionally masked so this document can be shared safely.

## Test Summary

| # | Scenario | Test Domain | Result | Observed RootCause | Status |
| --- | --- | --- | --- | --- | --- |
| 1 | Invalid ARN input | N/A | Passed | `INVALID_INPUT` | Complete |
| 2 | ACM API error | N/A | Passed | `API_ERROR` | Complete |
| 3 | Email validation certificate | `emailvalidation.<test-domain>` | Passed | `NOT_DNS` | Complete |
| 4 | Missing ACM CNAME | `missing.<test-domain>` | Passed | `CNAME_MISSING` | Complete |
| 5 | CNAME value mismatch | `mismatch.<test-domain>` | Passed | `CNAME_MISMATCH` | Complete |
| 6 | Issued certificate | `happy.<test-domain>` | Passed | `NOT_PENDING` | Complete |
| 7 | TXT conflict at ACM validation name | `txt.<test-domain>` | Passed | `TXT_CONFLICT` | Complete |
| 8 | CAA blocking | `caa.<test-domain>` | Passed | `CAA_BLOCKING` | Complete |
| 9 | Multi-domain/SAN certificate | `san-*.<test-domain>` | Passed | Multiple | Complete |

## Completed Test Details

### 1. Invalid ARN Input

Input used a malformed ACM ARN.

Observed output:

```json
{
  "RootCause": "INVALID_INPUT",
  "Summary": "Invalid ARN format",
  "Remediation": "Format: arn:aws:acm:<region>:<account>:certificate/<id>"
}
```

Result: Passed. The checker rejected the input before calling ACM.

### 2. ACM API Error

Input used a valid-looking ARN that could not be described, or the caller lacked the required ACM permission.

Observed output:

```json
{
  "RootCause": "API_ERROR",
  "Summary": "DescribeCertificate failed. Check permissions/region.",
  "Remediation": "Ensure the execution role has acm:DescribeCertificate and the ARN region is correct."
}
```

Result: Passed. The checker returned an actionable API error instead of continuing with DNS checks.

### 3. Email Validation Certificate

Certificate:

```text
arn:aws:acm:eu-west-1:<account-id>:certificate/<certificate-id>
```

Observed output:

```json
{
  "RootCause": "NOT_DNS",
  "Summary": "Uses EMAIL, not DNS.",
  "Remediation": "This tool only supports DNS validation.",
  "CertificateArn": "arn:aws:acm:eu-west-1:<account-id>:certificate/<certificate-id>",
  "Status": "PENDING_VALIDATION",
  "Domain": "emailvalidation.<test-domain>"
}
```

Result: Passed. The checker correctly stopped because the certificate uses email validation.

### 4. Missing ACM CNAME

Certificate:

```text
arn:aws:acm:eu-west-1:<account-id>:certificate/<certificate-id>
```

Observed output:

```json
{
  "RootCause": "CNAME_MISSING",
  "Summary": "CNAME not found at <acm-validation-name>.missing.<test-domain>.",
  "AllIssues": "CNAME_MISSING(missing.<test-domain>)",
  "Remediation": "Create CNAME: <acm-validation-name>.missing.<test-domain>. -> <acm-validation-target>.acm-validations.aws.",
  "CertificateArn": "arn:aws:acm:eu-west-1:<account-id>:certificate/<certificate-id>",
  "Status": "PENDING_VALIDATION",
  "Domain": "missing.<test-domain>",
  "IsFailed": "False",
  "IsRenewal": "False"
}
```

Result: Passed. The checker identified the missing public CNAME and returned the required CNAME target.

### 5. CNAME Value Mismatch

Certificate:

```text
arn:aws:acm:eu-west-1:<account-id>:certificate/<certificate-id>
```

Observed output:

```json
{
  "RootCause": "CNAME_MISMATCH",
  "Summary": "CNAME value does not exactly match. Expected=<expected-acm-validation-target>, Actual=<wrong-acm-validation-target>",
  "AllIssues": "CNAME_MISMATCH(mismatch.<test-domain>) | TXT_CONFLICT(mismatch.<test-domain>)",
  "Remediation": "Verify CNAME <acm-validation-name>.mismatch.<test-domain>. exactly matches ACM expected value <expected-acm-validation-target>. Current value: <wrong-acm-validation-target> | Verify TXT record at the exact ACM CNAME validation name for mismatch.<test-domain>. If it is in the same hosted zone, remove it so the CNAME can exist alone.",
  "CertificateArn": "arn:aws:acm:eu-west-1:<account-id>:certificate/<certificate-id>",
  "Status": "PENDING_VALIDATION",
  "Domain": "mismatch.<test-domain>",
  "IsFailed": "False",
  "IsRenewal": "False"
}
```

Result: Passed. The checker selected `CNAME_MISMATCH` as the primary root cause and preserved the additional `TXT_CONFLICT` in `AllIssues`.

### 6. Issued Certificate

Certificate:

```text
arn:aws:acm:eu-west-1:<account-id>:certificate/<certificate-id>
```

Observed output:

```json
{
  "RootCause": "NOT_PENDING",
  "Summary": "Certificate is ISSUED.",
  "Remediation": "No DNS validation action is needed for this status.",
  "CertificateArn": "arn:aws:acm:eu-west-1:<account-id>:certificate/<certificate-id>",
  "Status": "ISSUED",
  "Domain": "happy.<test-domain>"
}
```

Result: Passed. The checker correctly skipped DNS diagnostics because the certificate is already issued.

### 7. TXT Conflict at ACM Validation Name

Certificate:

```text
arn:aws:acm:eu-west-1:<account-id>:certificate/<certificate-id>
```

Observed output:

```json
{
  "RootCause": "TXT_CONFLICT",
  "Summary": "TXT record exists at the exact ACM CNAME validation name <acm-validation-name>.txt.<test-domain>.. Verify whether it is in the same hosted zone before removing it. TXT values: [\"<txt-value>\"]",
  "AllIssues": "TXT_CONFLICT(txt.<test-domain>)",
  "Remediation": "Verify TXT record at the exact ACM CNAME validation name for txt.<test-domain>. If it is in the same hosted zone, remove it so the CNAME can exist alone.",
  "CertificateArn": "arn:aws:acm:eu-west-1:<account-id>:certificate/<certificate-id>",
  "Status": "PENDING_VALIDATION",
  "Domain": "txt.<test-domain>",
  "IsFailed": "False",
  "IsRenewal": "False"
}
```

Result: Passed.

Note: `TXT_CONFLICT` is valid only when a TXT record exists at the exact ACM validation owner name. TXT records returned from the CNAME target under `acm-validations.aws` are expected ACM-side records and must not be treated as customer-side TXT conflicts.

For the earlier TXT-only state before creating the CNAME, the expected primary root cause remains `CNAME_MISSING`, with `TXT_CONFLICT` included in `AllIssues`.

### 8. CAA Blocking

Status: Complete.

Test setup:

1. Create or use `caa.<test-domain>`.
2. Add a restrictive CAA record that does not allow Amazon CA, for example:

```text
0 issue "digicert.com"
```

3. Request an ACM DNS-validation certificate for `caa.<test-domain>`.
4. Create the correct ACM validation CNAME.
5. Run the checker before changing the CAA record.

Observed result:

```text
RootCause = CAA_BLOCKING
```

Result: Passed. The checker detected that the restrictive CAA record did not authorize Amazon CA issuance after the correct ACM validation CNAME was created.

Cleanup reminder:

```text
Remove or update the restrictive CAA record after the test.
```

### 9. Multi-Domain/SAN Certificate

Status: Complete.

Test setup:

1. Request one ACM certificate with multiple names, for example:

```text
san-main.<test-domain>
san-missing.<test-domain>
san-mismatch.<test-domain>
```

2. Create the correct ACM CNAME for at least one SAN.
3. Leave one SAN validation CNAME missing.
4. Optionally create an incorrect CNAME target for another SAN.
5. Run the checker against the single SAN certificate ARN.

Observed result:

```text
All DomainValidationOptions are checked independently.
AllIssues includes each affected SAN domain.
Primary RootCause follows the configured priority order.
```

Result: Passed. The checker iterated over all `DomainValidationOptions` in the SAN certificate and reported issues per affected domain instead of checking only the primary domain.

Validation points:

| Check | Result |
| --- | --- |
| Domain iteration | Passed. Every SAN entry was evaluated, not only the primary domain. |
| CNAME checks | Passed. Each SAN had independent CNAME existence and value validation. |
| TXT checks | Passed. TXT conflict detection was evaluated per SAN validation name. |
| CAA checks | Passed. CAA was evaluated per SAN domain. |
| NS checks | Passed for loop behavior. Shared base domains are checked once, not once per SAN. |
| Output | Passed. `AllIssues` identified affected domains individually. |

## Pending Tests

### NS Delegation Inconsistency

Status: Pending.

This scenario requires a safe disposable domain or delegated test zone where parent delegation and authoritative NS records can be intentionally mismatched without impacting a real production domain.

## Open Follow-Ups

| Item | Status | Notes |
| --- | --- | --- |
| NS delegation inconsistency test | Pending | Requires a safe disposable domain or delegated test zone where parent delegation and authoritative NS records can be intentionally mismatched. |
| TXT conflict flow | Done | `TXT_CONFLICT` is reported only for TXT records at the exact ACM validation owner name. TXT records returned from the CNAME target under `acm-validations.aws` are ignored. |
| Public-safe test data | Done | Account ID, certificate IDs, and validation tokens are masked in this document |
