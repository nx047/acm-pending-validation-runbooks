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
| 7 | TXT conflict at ACM validation name | `txt.<test-domain>` | Passed with note | `TXT_CONFLICT` | Complete |
| 8 | CAA blocking | `caa.<test-domain>` | Not tested | N/A | Pending |
| 9 | Multi-domain/SAN certificate | TBD | Not tested | N/A | Pending |

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

Result: Passed with note.

Note: This result represents the state after the ACM validation CNAME exists while a TXT record still remains at the exact same validation name. In that state, TXT_CONFLICT is expected as the primary root cause so the remaining TXT record can be removed. For the earlier TXT-only state before creating the CNAME, the expected primary root cause remains CNAME_MISSING, with TXT_CONFLICT included in AllIssues.


## Pending Tests

### CAA Blocking

Planned setup:

1. Create or use `caa.<test-domain>`.
2. Add a restrictive CAA record that does not allow Amazon CA, for example:

```text
0 issue "digicert.com"
```

3. Request an ACM DNS-validation certificate for `caa.<test-domain>`.
4. Create the correct ACM validation CNAME.
5. Run the checker before changing the CAA record.

Expected result:

```text
RootCause = CAA_BLOCKING
```

Cleanup:

```text
Remove or update the restrictive CAA record after the test.
```

### Multi-Domain/SAN Certificate

Planned setup:

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

Expected result:

```text
All DomainValidationOptions are checked independently.
AllIssues includes each affected SAN domain.
Primary RootCause follows the configured priority order.
```

Validation points:

| Check | Expected Behavior |
| --- | --- |
| Domain iteration | Every SAN entry is evaluated, not only the primary domain |
| CNAME checks | Each SAN has independent CNAME existence and value validation |
| TXT checks | TXT conflicts are reported per SAN validation name |
| CAA checks | CAA is evaluated for each SAN domain |
| NS checks | Shared base domains are checked once, not once per SAN |
| Output | `AllIssues` identifies the affected domain for each issue |

## Open Follow-Ups

| Item | Status | Notes |
| --- | --- | --- |
| CAA blocking test | Pending | Requires restrictive CAA record and a DNS-correct ACM CNAME |
| Multi-domain/SAN test | Pending | Requires one certificate containing multiple validation domains |
| TXT conflict flow | Done | TXT-only before CNAME creation is expected to report `CNAME_MISSING` as the primary root cause with `TXT_CONFLICT` in `AllIssues`. After the ACM CNAME is created, if TXT still remains at the same validation name, `TXT_CONFLICT` is expected as the primary root cause. |
| Public-safe test data | Done | Account ID, certificate IDs, and validation tokens are masked in this document |