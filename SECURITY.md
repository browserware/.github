# Security Policy

## Supported Versions

Browserware is in early development (`< v1.0`). We will provide security updates for the current development branch but do not make stability or security guarantees at this stage.

## Reporting Vulnerabilities

If you discover a security vulnerability, please **do not open a public issue**.

### How to Report

Send an email to: **security@guria.dev**

Please include:
- Description of the vulnerability
- Steps to reproduce
- Impact assessment
- Suggested fix (if known)

### What to Expect

- We will acknowledge receipt within 48 hours
- We will provide a timeline for resolution and disclosure
- We will work with you to understand and fix the issue
- We will disclose publicly once a fix is released

## Security Scope

We consider the following to be in-scope for security reports:

- Remote code execution through malicious URLs
- Privilege escalation via browser launching
- Exposure of sensitive data (profiles, credentials, browsing history)
- Default handler hijacking or redirect abuse
- Local privilege escalation through system integration

### Out of Scope

- Browser vulnerabilities unrelated to Browserware
- Issues requiring physical access or already-compromised systems
- Denial-of-service via excessive resource usage
- Issues in third-party dependencies (please report upstream)

## Security Best Practices

Users of Browserware should:

1. **Review rules before applying** - Ensure routing rules match your intent
2. **Audit command construction** - Use `--dry-run` or inspect flags before registering as default browser
3. **Keep updated** - Pull latest releases for security patches
4. **Understand permissions** - Browserware requires elevated privileges for default browser registration

## Security Principles

Our approach to security:

- **Explicit over implicit** - No hidden URL transformations or redirects
- **Fail closed** - Errors prevent routing rather than falling back to unsafe behavior
- **Transparent commands** - All browser launches are inspectable
- **Minimal privilege** - Request only permissions necessary for functionality
- **Input validation** - Strict validation of URLs, paths, and configuration

## Disclosure Policy

We follow coordinated vulnerability disclosure:

1. Report → Triaged → Fixed privately
2. Release → Public disclosure with credit to reporter
3. Timeline typically: 7-14 days for critical issues, 30+ days for complex issues

Questions about security policy: **security@guria.dev**
