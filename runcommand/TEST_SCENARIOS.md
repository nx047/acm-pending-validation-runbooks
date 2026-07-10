# ACM DNS Validation Checker Test Scenarios

This document describes manual test scenarios for the Run Command version of the ACM DNS Validation Checker.

The examples use `<your-domain>` as a placeholder so the document can be safely published to GitHub.

## DNS Setup

| Item | Value |
| --- | --- |
| DNS provider | Any public authoritative DNS provider |
| Domain placeholder | `<your-domain>` |
| Test style | One certificate per test subdomain |
| Cloudflare proxy behavior | Use DNS-only records for ACM validation CNAMEs if Cloudflare proxy is enabled |
| DNS record changes | Create records in the authoritative DNS provider for your domain |

Before testing, confirm which nameservers are authoritative for the domain:

```bash
dig NS <your-domain> +short
```

Create and modify test records in the DNS provider shown by the public delegation. Route 53 is convenient for testing because records can be managed with AWS CLI, but it is not required.

## Scenario Matrix

| # | Scenario | Certificate Domain | DNS Setup | Expected Result |
| --- | --- | --- | --- | --- |
| 1 | Invalid input | N/A | No DNS setup needed | `INVALID_INPUT` |
| 2 | API error | N/A | Use a valid-looking ACM ARN that does not exist or cannot be described | `API_ERROR` |
| 3 | Not DNS validation | `email.<your-domain>` | Request an ACM certificate with email validation | `NOT_DNS` |
| 4 | Missing CNAME | `missing.<your-domain>` | Request the certificate, but do not create the ACM validation CNAME | `CNAME_MISSING` |
| 5 | CNAME mismatch | `mismatch.<your-domain>` | Create the ACM validation CNAME name with an intentionally wrong target value | `CNAME_MISMATCH` |
| 6 | TXT conflict | `txt.<your-domain>` | Create a TXT record at the exact ACM validation CNAME name instead of creating the CNAME | `RootCause=CNAME_MISSING`, `AllIssues` includes `TXT_CONFLICT` |
| 7 | CAA blocking | `caa.<your-domain>` | Add a restrictive CAA record, then create the correct ACM validation CNAME | `CAA_BLOCKING` |
| 8 | No issue found | `pending-ok.<your-domain>` | Create the exact ACM validation CNAME, then run the checker before ACM changes the certificate to `ISSUED` | `NO_ISSUE_FOUND` |
| 9 | Happy path | `happy.<your-domain>` | Create the exact ACM validation CNAME and wait until the certificate is issued | `NOT_PENDING` |

## Test Subdomains

| Purpose | Subdomain |
| --- | --- |
| Missing CNAME test | `missing.<your-domain>` |
| CNAME mismatch test | `mismatch.<your-domain>` |
| TXT conflict test | `txt.<your-domain>` |
| CAA blocking test | `caa.<your-domain>` |
| Pending but DNS-correct test | `pending-ok.<your-domain>` |
| Successful validation test | `happy.<your-domain>` |

## Scenario Details

### 1. Invalid Input

Run the automation with a broken ACM ARN.

```text
CertificateArn = broken-arn
```

Expected result:

```text
RootCause = INVALID_INPUT
```

### 2. API Error

Run the automation with an ARN that has a valid ACM ARN shape but cannot be described.

```text
CertificateArn = arn:aws:acm:<region>:<account-id>:certificate/00000000-0000-0000-0000-000000000000
```

Expected result:

```text
RootCause = API_ERROR
```

### 3. Not DNS Validation

Request an ACM certificate using email validation.

Expected result:

```text
RootCause = NOT_DNS
```

### 4. Missing CNAME

Request an ACM certificate for:

```text
missing.<your-domain>
```

Do not create the ACM DNS validation CNAME in your authoritative DNS provider.

Expected result:

```text
RootCause = CNAME_MISSING
```

### 5. CNAME Mismatch

Request an ACM certificate for:

```text
mismatch.<your-domain>
```

In your authoritative DNS provider, create the ACM validation CNAME record name, but use a wrong target value.

Expected result:

```text
RootCause = CNAME_MISMATCH
```

### 6. TXT Conflict

Request an ACM certificate for:

```text
txt.<your-domain>
```

Do not create the ACM validation CNAME. Instead, create a TXT record at the exact ACM validation CNAME name.

Example:

```text
Name = <ACM_CNAME_NAME>
Type = TXT
Value = "test-conflict"
```

Expected result:

```text
RootCause = CNAME_MISSING
AllIssues includes TXT_CONFLICT
```

Note: A DNS name cannot normally have a CNAME and TXT record at the same exact owner name at the same time. Because the CNAME is missing, `CNAME_MISSING` remains the primary root cause, while `TXT_CONFLICT` is reported as an additional issue.

### 7. CAA Blocking

Add a restrictive CAA record for:

```text
caa.<your-domain>
```

Example CAA record:

```text
0 issue "digicert.com"
```

Then request an ACM certificate for:

```text
caa.<your-domain>
```

Create the correct ACM validation CNAME in your authoritative DNS provider.

Expected result:

```text
RootCause = CAA_BLOCKING
```

After this test, remove or update the restrictive CAA record if it is no longer needed.

### 8. No Issue Found

Request an ACM certificate for:

```text
pending-ok.<your-domain>
```

Create the exact ACM validation CNAME record provided by ACM, then run the checker while the certificate is still `PENDING_VALIDATION`.

Expected result:

```text
RootCause = NO_ISSUE_FOUND
```

This means the DNS records look correct, but ACM has not completed validation yet. This is usually a timing or propagation state.

### 9. Happy Path

Request an ACM certificate for:

```text
happy.<your-domain>
```

In your authoritative DNS provider, create the exact ACM validation CNAME record provided by ACM. Wait until the certificate status becomes `ISSUED`.

Expected result:

```text
RootCause = NOT_PENDING
```

This means the certificate is already `ISSUED` or otherwise no longer pending, so the checker exits before running detailed DNS issue checks.

## Recommended Test Order

| Order | Scenario |
| --- | --- |
| 1 | `INVALID_INPUT` |
| 2 | `API_ERROR` |
| 3 | `NOT_DNS` |
| 4 | `CNAME_MISSING` |
| 5 | `CNAME_MISMATCH` |
| 6 | `TXT_CONFLICT` |
| 7 | `CAA_BLOCKING` |
| 8 | `NO_ISSUE_FOUND` |
| 9 | Happy path |

## Additional Considerations

| Case | Notes |
| --- | --- |
| `TOOL_MISSING` | Can be tested by running on a target instance without `dig`, but this is usually a prerequisite validation rather than a DNS scenario. |
| `NS_INCONSISTENCY` | Requires intentionally breaking or mismatching delegation, so test only with a disposable domain or separate hosted zone. |
| `RENEWAL_FAILURE` | Treat as an additional consideration. It appears when ACM returns `RenewalSummary`, usually for an existing certificate in renewal flow where DNS validation records may have changed. |

## Notes

- Use separate ACM certificates for each test subdomain to keep the scenarios isolated.
- Keep the real domain name, AWS account ID, certificate ARN, and hosted zone ID out of public screenshots and logs.
- If a failed ACM certificate reaches a terminal `FAILED` state, request a new certificate after fixing DNS.
- If using Cloudflare, ACM validation CNAME records should be DNS-only. Proxied records can hide the original CNAME target from public DNS lookups.
